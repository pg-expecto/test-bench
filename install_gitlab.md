# Установка GitLab CE на виртуальную машину Debian

Сводная инструкция по развёртыванию GitLab Community Edition через официальный Omnibus-пакет. Стилистика и контекст — как в инструкции по AWX: пользователь `test`, хост `management2`, IP `192.168.0.111`, Debian 12. В отличие от AWX, здесь Kubernetes не нужен — GitLab ставится как единый пакет со всеми сервисами внутри (Omnibus) [```docs.gitlab.com```](https://docs.gitlab.com/install/package/debian/).

## Требования

- Debian 12 (Bookworm), x86_64 или arm64 [```docs.gitlab.com```](https://docs.gitlab.com/install/package/)
- Минимум 4 vCPU и 4 ГБ RAM [```rosehosting.com```](https://www.rosehosting.com/blog/how-to-install-gitlab-on-debian-12/); для комфортной работы рекомендую 8 ГБ, особенно если на этой же ВМ живёт K3s с AWX
- **20+ ГБ свободного диска**: сам пакет после распаковки занимает около 3.1 ГБ [```wiki.crowncloud.net```](https://wiki.crowncloud.net/?How_to_Install_Gitlab_On_Debian_12), плюс растущие репозитории и базы
- Доступ к `packages.gitlab.com` и `storage.googleapis.com` — пакеты хранятся в Google Cloud Storage [```docs.gitlab.com```](https://docs.gitlab.com/install/package/debian/). Учитывая ваш опыт с обрывами при скачивании образов AWX, будьте готовы к повторным попыткам — пакет весит ~1.3 ГБ

## Шаг 1. Подготовка системы

```bash
sudo apt update && sudo apt upgrade -y

# Зависимости
sudo apt install -y curl openssh-server ca-certificates apt-transport-https gnupg2

# SSH-сервер должен работать
sudo systemctl enable --now ssh

# Открыть порты (если брандмауэр активен)
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

Порты 22, 80 и 443 нужны для SSH-доступа к репозиториям, веб-интерфейса и HTTPS соответственно [```docs.gitlab.com```](https://docs.gitlab.com/install/package/debian/).

## Шаг 2. Подключение репозитория GitLab

```bash
curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
```

Скрипт добавит GPG-ключ GitLab, создаст файл репозитория в `/etc/apt/sources.list.d/` и обновит список пакетов [```ultravds.com```](https://ultravds.com/blog/kak-ustanovit-gitlab-na-debian).

> **Почему `gitlab-ce`, а не `gitlab-ee`:** CE — бесплатная Community Edition. Из России репозитории EE недоступны, CE работает — используйте именно её [```serveradmin.ru```](https://serveradmin.ru/ustanovka-i-nastrojka-gitlab/).

## Шаг 3. Установка GitLab CE

Определите внешний адрес заранее. Если доменного имени нет (как в вашем случае — ВМ с IP), используйте IP напрямую:

```bash
sudo EXTERNAL_URL="http://192.168.0.111" apt install gitlab-ce -y
```

> **Важно:** с IP-адресом используйте только `http://`. Вариант `https://` требует валидного доменного имени — Let's Encrypt не выпускает сертификаты на IP, и установка с таким URL упадёт при запросе сертификата.

Установка займёт несколько минут: пакет скачается (~1.3 ГБ), распакуется, и GitLab автоматически сконфигурируется на указанный адрес [```docs.gitlab.com```](https://docs.gitlab.com/install/package/debian/). В конце увидите:

```
gitlab Reconfigured!
```

## Шаг 4. Первый вход

GitLab генерирует случайный пароль для администратора `root` и сохраняет его в файл `/etc/gitlab/initial_root_password` [```docs.gitlab.com```](https://docs.gitlab.com/install/package/debian/):

```bash
sudo cat /etc/gitlab/initial_root_password
```

> ⚠️ **Файл с паролем автоматически удаляется при первом `reconfigure` спустя 24 часа** — сохраните пароль сразу или смените его после входа [```docs.gitlab.com```](https://docs.gitlab.com/install/package/debian/).

Вход: `http://192.168.0.111`, логин `root`, пароль из файла. Сразу после входа смените пароль и email в профиле.

Если пароль потерялся, его можно сбросить [```atlantic.net```](https://www.atlantic.net/dedicated-server-hosting/how-to-set-up-gitlab-continuous-integration-ci-pipelines-on-debian-12/):

```bash
sudo gitlab-rake 'gitlab:password:reset[root]'
```

## Шаг 5. Повседневное управление

```bash
sudo gitlab-ctl status      # состояние всех сервисов (postgres, redis, nginx, puma...)
sudo gitlab-ctl restart     # перезапуск
sudo gitlab-ctl stop        # остановка
sudo gitlab-ctl tail        # просмотр логов в реальном времени
```

Повторное применение конфигурации после правок `/etc/gitlab/gitlab.rb`:

```bash
sudo gitlab-ctl reconfigure
```

## Известные проблемы и решения

### Медленно или обрывается скачивание пакета

Как и в случае с `awx-ee`, большие файлы из-за рубежа могут рваться. apt умеет докачивать — просто повторите `apt install gitlab-ce -y`. Если `packages.gitlab.com` недоступен совсем, запасной путь — скачать конкретный `.deb` вручную со страницы пакета на `packages.gitlab.com` и установить через `apt install ./gitlab-ce_*.deb`.

### Ошибка 502 в веб-интерфейсе сразу после установки

Норма: сервисы ещё стартуют. Подождите 2–3 минуты и обновите страницу. Если не уходит — `sudo gitlab-ctl status` покажет, какой сервис не поднялся, а `sudo gitlab-ctl tail` — почему.

### GitLab «тормозит» на ВМ с AWX

GitLab и AWX вместе просят прилично памяти. Если у ВМ меньше 8 ГБ, добавьте swap:

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## Справочник по диагностике

| Симптом | Причина | Действие |
|---|---|---|
| `apt install` рвётся на середине | нестабильный канал | повторить команду — apt докачает |
| `502 Whoops` в браузере | сервисы ещё стартуют | подождать 2–3 мин, затем `gitlab-ctl status` |
| пароль из `initial_root_password` не подходит | файл старше 24 ч или пароль сменён | `gitlab-rake 'gitlab:password:reset[root]'` |
| установка просит домен при `https://` | Let's Encrypt не работает с IP | использовать `http://<IP>` |
| веб не открывается извне | закрыт порт 80/443 | `ufw allow 80/tcp; ufw allow 443/tcp` |

## Удаление (если нужно начать заново)

```bash
sudo apt remove gitlab-ce
sudo apt purge gitlab-ce
sudo rm -rf /etc/gitlab /var/opt/gitlab /var/log/gitlab
```

Логичный следующий шаг в вашей связке — установка `gitlab-runner` на этой же ВМ для CI/CD и интеграция GitLab с уже поднятым AWX (запуск Ansible-плейбуков из пайплайнов через API). Скажите, когда дойдёте — подготовлю инструкцию и под эту часть 🙂
