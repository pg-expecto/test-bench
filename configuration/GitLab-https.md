# Настройка HTTPS-доступа к GitLab (management2, 192.168.0.111)

## 1. Исходная инфраструктура

| Компонент | Узел | IP | Аккаунт | Примечание |
|---|---|---|---|---|
| GitLab (Omnibus) | management2 | 192.168.0.111 | test / root (админ GitLab) | Дистрибутив Debian |
| PostgreSQL 1 | psql1 | 192.168.0.105 | test | — |
| PostgreSQL 2 | psql2 | 192.168.0.108 | test | — |
| AWX | management2 (k3s) | 192.168.0.111:32264 | admin | NodePort в кластере k3s |
| Клиент | Windows-ВМ | — | — | Браузерный доступ к GitLab |

## 2. Диагностика исходного состояния

**Тип установки.** GitLab развёрнут как Omnibus-пакет (нативно, без Docker и Kubernetes): подтверждено активным сервисом `gitlab-runsvdir.service` и отсутствием контейнеров. Все компоненты (nginx, puma, workhorse, gitaly, postgresql, redis, sidekiq) работают под supervision runsvdir.

**Конфигурация HTTPS в `/etc/gitlab/gitlab.rb` на момент проверки:**

| Параметр | Значение | Оценка |
|---|---|---|
| `external_url` | `https://gitlab.local` | URL объявлен как HTTPS |
| `letsencrypt['enable']` | `false` | LE отключён явно (для приватного IP он и не выдаётся) |
| `nginx['ssl_certificate']` / `..._key` | закомментированы | Сертификат не настроен |
| `nginx['redirect_http_to_https']` | `true`, затем переопределён на `false` | Действует последнее присвоение |
| `nginx['listen_port']` | `8080` | Порт, совпадающий с дефолтным портом puma |
| `nginx['listen_https']` | `false` | TLS на bundled nginx выключен |

**Фактическое состояние.** Порт 443 не слушался ни одним процессом; HTTP-доступ к GitLab отвечал `502 Bad Gateway`.

## 3. Выявленные дефекты и их причины

1. **Кольцевая петля на порту 8080.** После правки `gitlab.rb` bundled nginx слушал `0.0.0.0:8080`, но puma и workhorse используют `localhost:8080` как внутренний бэкенд. Запрос попадал в nginx, который проксировал сам на себя, — отсюда постоянный 502.
2. **Устаревшие ключи конфигурации.** В GitLab 19.2+ ключи `nginx['listen_port']`, `nginx['listen_https']`, `nginx['redirect_http_to_https']` объявлены deprecated; Chef игнорировал их и сообщал `0/911 resources updated` — reconfigure ничего не менял.
3. **Незавершённая схема ingress.** В конце `gitlab.rb` присутствовал блок «HTTPS consolidation via ingress-nginx», но контроллер был только что установлен, Ingress-ресурсов не существовало (`kubectl get ingress -A` → No resources found).
4. **Отсутствие сертификата** — ни Let's Encrypt, ни собственного.

## 4. Выполненные изменения

### 4.1 Исправление конфигурации GitLab

В конец `/etc/gitlab/gitlab.rb` добавлен блок с актуальными ключами; старые закомментированы:

```ruby
# --- HTTPS consolidation via ingress-nginx (fixed keys) ---
gitlab_rails['nginx']['listen_port'] = 8090
gitlab_rails['nginx']['listen_https'] = false
gitlab_rails['nginx']['redirect_http_to_https'] = false
gitlab_rails['nginx']['proxy_set_headers'] = {
  "X-Forwarded-Proto" => "https",
  "X-Forwarded-Ssl" => "on"
}
```

- Bundled nginx переведён на порт **8090** (устранение петли с puma);
- `X-Forwarded-Proto` — чтобы Rails за ingress корректно считал схему HTTPS;
- Применено: `gitlab-ctl reconfigure` (3/912 resources updated) + `gitlab-ctl restart nginx`.

### 4.2 Генерация самоподписанного сертификата

```bash
sudo mkdir -p /etc/gitlab/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/gitlab/ssl/gitlab.local.key \
  -out /etc/gitlab/ssl/gitlab.local.crt \
  -subj "/CN=gitlab.local" \
  -addext "subjectAltName=DNS:gitlab.local,IP:192.168.0.111"
```

### 4.3 Развертывание ingress-части в k3s

- ingress-nginx установлен через Helm: под `ingress-nginx-controller` Running, Service типа LoadBalancer с адресом `192.168.0.111`, порты `80:31863/TCP`, `443:31835/TCP`;
- создан TLS-секрет `gitlab-tls` из сертификата;
- созданы Service `gitlab-external` и Endpoints (адрес `192.168.0.111:8090`) — Service без селектора, т.к. GitLab работает вне кластера;
- создан Ingress `gitlab` с правилом `host: gitlab.local`, TLS-секретом `gitlab-tls`, `ingressClassName: nginx`;
- проверка: `kubectl get ingress -A` → адрес `192.168.0.111`, порты `80, 443`.

### 4.4 DNS-запись

На management2 и на Windows-клиенте: `192.168.0.111 gitlab.local` (в `/etc/hosts` и в `C:\Windows\System32\drivers\etc\hosts` соответственно).

## 5. Результаты проверки

| Проверка | До настройки | После настройки |
|---|---|---|
| `curl -I -H "Host: gitlab.local" http://...:8090` | — (порт закрыт, на 8080 — 502) | `HTTP/1.1 302 Found → https://gitlab.local/users/sign_in` |
| `curl -I https://gitlab.local` | соединение не устанавливалось | TLS-рукопожатие прошло; отказ только из-за недоверия к самоподписанному серту |
| `curl -kI https://gitlab.local` | — | `HTTP/2 302 → https://gitlab.local/users/sign_in` |

Схема трафика по факту:

```
Браузер → :443 ingress-nginx (TLS-терминация, cert gitlab-tls)
        → Service gitlab-external
        → 192.168.0.111:8090 (bundled nginx, plain HTTP)
        → gitlab-workhorse → puma (127.0.0.1:8080)
```

## 6. Доверие к сертификату на клиентах

**management2 (Linux):**

```bash
sudo cp /etc/gitlab/ssl/gitlab.local.crt /usr/local/share/ca-certificates/gitlab.local.crt
sudo update-ca-certificates
```

**Windows-клиент (PowerShell от имени администратора):**

```powershell
# hosts
Add-Content -Path C:\Windows\System32\drivers\etc\hosts -Value "`n192.168.0.111 gitlab.local" -Encoding ascii
# сертификат (после копирования файла с management2)
Import-Certificate -FilePath C:\Users\test\gitlab.local.crt -CertStoreLocation Cert:\LocalMachine\Root
# проверка
curl.exe -I https://gitlab.local
```

Ожидание: замочек в браузере закрытый, `HTTP/2 302` на страницу входа без предупреждений. Для Firefox — отдельный импорт в его собственное хранилище сертификатов.

## 7. Остаточные риски и рекомендации

1. **Срок сертификата — 365 дней.** Перевыпуск: повторный `openssl req ...` + обновление секрета `kubectl create secret tls gitlab-tls -n ingress-nginx --cert=... --key=... --dry-run=client -o yaml | kubectl apply -f -`. Рекомендуется оформить автоматизацию (cron или Ansible-роль в AWX).
2. **HTTP на порту 80** ingress отдаёт 404 — редирект с HTTP на HTTPS не проверен. При необходимости включается аннотацией `nginx.ingress.kubernetes.io/ssl-redirect: "true"`.
3. **Устаревший ключ `v1 Endpoints`** — работает, но в k8s 1.33+ рекомендуется миграция на `EndpointSlice` (некритично).
4. **Масштабирование на клиентов** — запись в hosts и импорт серта нужно повторить на каждой машине; в доменной среде сертификат можно раздать через GPO.
5. **Воспроизводимость** — все шаги целесообразно оформить Ansible-ролью и запустить через AWX, чтобы настройка была повторяемой на всех узлах стенда.

## 8. Статус задачи

✅ **Выполнено и проверено:** HTTPS-доступ к GitLab по адресу `https://gitlab.local` работает — TLS терминируется ingress-nginx, GitLab отвечает редиректом на страницу входа, петля портов устранена, конфигурация приведена к актуальному синтаксису. Осталась финальная проверка из браузера Windows после импорта сертификата.
