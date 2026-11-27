# Финальный отчёт: сборка pipeline GitLab → AWX → psql-узлы

**Дата:** 27.11.2026
**Сервер управления:** management2 (пользователь test)
**Зона охвата:** кластер Kubernetes (namespace awx), GitLab (gitlab.local, 192.168.0.111), узлы psql1 (192.168.0.105) и psql2 (192.168.0.108)

---

## 1. Статус кластера

| Компонент | Объект | Статус | Комментарий |
|---|---|---|---|
| AWX operator | awx-operator-controller-manager | Running | 21 рестарт за срок жизни — характерно для operator'а, не критично |
| AWX CR | awx-demo (namespace awx) | Healthy | Версия 24.6.1, миграции пройдены |
| Поды AWX | awx-demo-web, awx-demo-task | 3/3 и 4/4 Running | Пересозданы после подключения CA-bundle (новые хеши deployment) |
| PostgreSQL | awx-demo-postgres-15-0 | 1/1 Running | Внутренний БД AWX, не трогался |
| CA-bundle | secret awx-demo-cabundle | Создан | Сертификат GitLab из /tmp/gitlab-ca.crt, подключён через bundle_cacert_secret |
| SSL-доверие GitLab | https://gitlab.local | Работает | Сертификат извлечён openssl s_client, доверие внутри контейнеров настроено |

**Особенности:**
- CA-бундл примонтируется init-контейнером `init-bundle-ca-trust` при старте пода — доверие действует во всех рабочих контейнерах (awx-demo-task, awx-demo-ee).
- Рабочие контейнеры task-пода: redis, awx-demo-task, awx-demo-ee, awx-demo-rsyslog. Git-синхронизация и playbook'и выполняются в awx-demo-ee.

---

## 2. Проверка SSH/sudo

| Узел | IP | Вход по ключу | sudo NOPASSWD | Проверка |
|---|---|---|---|---|
| psql1 | 192.168.0.105 | Да | Да (root) | ssh -o BatchMode=yes ... hostname → psql1; sudo -n whoami → root |
| psql2 | 192.168.0.108 | Да | Да (root) | ssh -o BatchMode=yes ... hostname → psql2; sudo -n whoami → root |

**Детали настройки:**
- Ключ: `~/.ssh/id_ed25519` (ed25519, без passphrase), права 600 — корректные.
- Ключ разложен на оба узла через `ssh-copy-id -i ~/.ssh/id_ed25519.pub test@<IP>`.
- sudoers: `/etc/sudoers.d/test` = `test ALL=(ALL) NOPASSWD: ALL`, права 440 — на обоих узлах.

---

## 3. Статус pipeline

| Этап | Объект | Статус |
|---|---|---|
| Репозиторий | GitLab, проект ansible-playbooks | Пушится с management2 (/test1/ansible-playbooks/ansible-playbooks) |
| Проект в AWX | ansible-playbooks from gitlab | Successful, Source Control Revision получен |
| Инвентарь | psql-nodes, source from-gitlab | Successful; hosts: psql1, psql2 (группа psql_nodes); Update on launch включён |
| Inventory файл | inventory/hosts.ini | С ansible_host=<IP>, ansible_user=test, ansible_become=true |
| Credential | psql-nodes-ssh (Machine) | Username test, SSH Private Key, Password пусто |
| Job Template | Deploy psql config | Создан; Launch прошёл успешно |
| Тестовый прогон | Пример задачи для psql-узлов | ok=3/ok=3, unreachable=0, failed=0 — узлы отдали Debian 13.8 |

**Особенности:**
- Update on launch: инвентарь автоматически обновляется из репозитория перед каждым запуском job'а.
- Зависимость от DNS устранена: в hosts.ini используются ansible_host=<IP> — playbook'и идут по IP, имена psql1/psql2 остаются только как логические метки.

---

## 4. Критичные проблемы — решённые (для истории)

1. **DNS-резолв psql1/psql2 на management2** — имена не резолвились; решено переходом на IP (ansible_host) и записью в /etc/hosts для ручных проверок.
2. **Конфликт host key на 192.168.0.108** — в known_hosts была запись с другим ключом; решено `ssh-keygen -R` + повторный ssh-copy-id с принятием нового ключа.
3. **Отсутствие ключа на psql2** — ssh-copy-id изначально остановился на первой команде (DNS-ошибка), ключ на psql2 не был разложен; решено повторным ssh-copy-id.
4. **sudo без NOPASSWD на psql2** — решено созданием /etc/sudoers.d/test.
5. **Парольная аутентификация в AWX** — credential шёл паролем вместо ключа (ошибка sshpass «Invalid/incorrect password»); решено пересозданием credential: ключ целиком, поле Password очищено.
6. **SSL-доверие GitLab** — решено через bundle_cacert_secret (AWX CR) вместо переменной GIT_SSL_NO_VERIFY.

---

## 5. Критичные моменты — требующие внимания (не решены)

| # | Проблема | Риск | Действие |
|---|---|---|---|
| 1 | Временная переменная GIT_SSL_NO_VERIFY ещё в AWX CR | Отключена проверка SSL в task-подах — маскирует реальные проблемы сертификата | Убрать block task_extra_env из CR (kubectl -n awx edit awx awx-demo) |
| 2 | Протестирован только демонстрационный playbook | Реальная настройка PostgreSQL не проверена | Разработать/прогнать рабочий playbook и проверить idempotency |
| 3 | Пароль пользователя test на psql2 был известен/использован при ssh-copy-id | Не все узлы унифицированы по доступу | Убедиться, что доступ только по ключу; при необходимости отключить PasswordAuthentication на узлах |
| 4 | В извест_hosts на management2 осталась запись от старого ключа (для 192.168.0.105 до смены) | Возможны будущие ложные конфликты при переустановках узлов | Периодически проверять known_hosts; для AWX не критично (HOST_KEY_CHECKING=False) |
| 5 | AWX-поды пересоздаются при каждом изменении CR | Простои веб-интерфейса на время rolling update | Планировать изменения CR вне рабочего времени |

---

## 6. Дальнейшие действия

1. **Уборка:** удалить task_extra_env с GIT_SSL_NO_VERIFY из AWX CR → поды пересоздадутся → убедиться, что проект по-прежнему синхронизируется (Projects → Sync).
2. **Наполнение playbook:** реальные задачи настройки PostgreSQL (пакеты, конфиги, БД и пользователи) — добавить в репозиторий, запушить, проверить Sync проекта и Update on launch.
3. **Idempotency:** второй прогон playbook'а должен давать changed=0 — обязательная проверка перед боевым применением.
4. **Отключить парольную аутентификацию SSH на psql-узлах** (после проверки, что всё работает по ключу) — снизит поверхность атаки.
5. **Бэкапы:** наладить регулярный backup AWX (postgres) и репозитория ansible-playbooks.

---

## 7. Чеклист готовности (итог)

- [x] Сертификат GitLab в CA-bundle AWX
- [x] Проект из GitLab синхронизируется в AWX
- [x] Инвентарь psql-nodes с хостами и переменными
- [x] SSH-ключ разложен на psql1 и psql2
- [x] sudo NOPASSWD на обоих узлах
- [x] Machine-credential в AWX (по ключу, без пароля)
- [x] Job Template создан и запущен успешно
- [x] Тестовый playbook: ok=3/3, failed=0, unreachable=0
- [ ] Убрать GIT_SSL_NO_VERIFY из CR
- [ ] Рабочий playbook по PostgreSQL (разработка и прогон)
- [ ] Отключить парольный вход SSH на узлах
- [ ] Бэкапы AWX и репозитория
