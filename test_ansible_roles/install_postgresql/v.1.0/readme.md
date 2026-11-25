# Автоматизированная установка PostgreSQL через GitLab и AWX

Ниже приведена сводная методика, полученная в ходе практического решения задачи на лабораторном стенде (management2 — узел управления с AWX на k3s; 192.168.0.105 — целевой сервер СУБД; 192.168.0.111 — GitLab). Инструкция отражает фактический порядок работ с учётом выявленных на стенде особенностей и препятствий.

## 1. Исходные условия

К началу работ предполагаются выполненными следующие предпосылки:

- сервер СУБД с Debian 13 доступен по SSH с узла управления (ключ `~/.ssh/id_ed25519`, пользователь `test` с правом `NOPASSWD: ALL` в sudoers);
- AWX развёрнут в кластере k3s и доступен по NodePort (`http://192.168.0.111:32264`);
- в GitLab по адресу `http://192.168.0.111` доступен namespace `test1`;
- пользователь GitLab аутентифицируется по HTTP посредством Personal Access Token (scope `write_repository`).

## 2. Формирование структуры проекта Ansible

Рабочая структура размещается в каталоге `/test/ansible-postgresql` и включает пять файлов:

```text
/test/ansible-postgresql/
├── install_postgres.yml
├── inventory/hosts.yml
└── roles/postgresql/
    ├── defaults/main.yml
    ├── handlers/main.yml
    └── tasks/main.yml
```

### 2.1. Inventory (inventory/hosts.yml)

Инвентарь объявляет один управляемый узел-псевдоним `awx_runner`; фактический адрес хоста передаётся в момент выполнения через переменную `db_server_ip` из survey AWX:

```yaml
all:
  hosts:
    awx_runner:
      ansible_host: "{{ db_server_ip }}"
      ansible_user: test
      ansible_become: true
      ansible_python_interpreter: /usr/bin/python3
```

### 2.2. Переменные по умолчанию роли (roles/postgresql/defaults/main.yml)

```yaml
pg_version: "17"
```

### 2.3. Задачи роли (roles/postgresql/tasks/main.yml)

Роль реализует установку PostgreSQL из официального репозитория PGDG (https://www.postgresql.org/download/linux/debian/) и включает следующие шаги:

1. Установка вспомогательных пакетов (`gnupg`, `ca-certificates`, `curl`), необходимых для подключения стороннего репозитория — модуль `apt`.
2. Создание каталога `/usr/share/postgresql-common/pgdg` и загрузка ключа подписи репозитория (`apt.postgresql.org/repo/apt-repo-release.asc`) — модули `file` и `get_url`.
3. Преобразование ключа в формат «dearmored» — модуль `command` (`dearmor`), с защитой от повтора исполнения (`changed_when: false`).
4. Добавление файла описания репозитория `/etc/apt/sources.list.d/pgdg.sources` — модуль `copy`.
5. Обновление кэша пакетов — модуль `apt` (`update_cache: true`).
6. Установка пакетов `postgresql-{{ pg_version }}` и `postgresql-client-{{ pg_version }}` — модуль `apt`.
7. Идемпотентная правка файла конфигурации `postgresql.conf`: установка `listen_addresses = '*'` — модуль `lineinfile`, с уведомлением обработчика перезапуска.
8. Аналогичная правка `pg_hba.conf`: открытие локальной сети (`host all all 0.0.0.0/0 scram-sha-256`) — модуль `lineinfile`, с уведомлением обработчика.
9. Финальная проверка доступности СУБД — модуль `command` (`pg_isready -h 127.0.0.1 -p 5432`).

Примечание: точечные правки `lineinfile` вместо перезаписи файлов целиком обеспечивают идемпотентность и сохраняют прочие параметры конфигурации PostgreSQL неизменными.

### 2.4. Обработчик (roles/postgresql/handlers/main.yml)

```yaml
- name: restart postgresql
  ansible.builtin.systemd:
    name: "postgresql@{{ pg_version }}-main"
    state: restarted
```

### 2.5. Сценарий (install_postgres.yml)

Плейбук динамически добавляет целевой хост в in-memory инвентарь и применяет роль:

```yaml
---
- name: Добавить целевой сервер СУБД во внутренний inventory
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Добавить хост по IP
      ansible.builtin.add_host:
        name: awx_runner
        ansible_host: "{{ db_server_ip }}"
        ansible_user: test
        ansible_become: true

- name: Установка PostgreSQL {{ pg_version }} на Debian 13
  hosts: awx_runner
  become: true
  roles:
    - postgresql
```

Схема данных корректна: `db_server_ip` и `pg_version` являются входными параметрами, передаваемыми извне (из survey AWX) в качестве extra vars.

## 3. Размещение проекта в GitLab

Работы выполняются в каталоге проекта на management2.

1. Инициализация репозитория:
   ```bash
   cd /test/ansible-postgresql
   git init
   ```
   При использовании Git версии до 2.28 ветка по умолчанию получает имя `master`; её следует переименовать:
   ```bash
   git branch -m master main
   ```
   Рекомендуется также установить ветку по умолчанию для всех будущих репозиториев:
   ```bash
   git config --global init.defaultBranch main
   ```
2. Привязка удалённого репозитория (плейсхолдеры `gitlab.example.com` и `devops` подлежат замене фактическими значениями):
   ```bash
   git remote add origin http://192.168.0.111/test1/ansible-postgresql.git
   ```
3. Настройка идентификации коммиттера (email должен совпадать с email профиля GitLab для корректной привязки коммитов к учётной записи):
   ```bash
   git config --global user.name "Ваше Имя"
   git config --global user.email "ваш-email"
   ```
4. Фиксация и публикация:
   ```bash
   git add .
   git commit -m "Initial commit: PostgreSQL role for Debian 13"
   git push -u origin main
   ```
   При запросе аутентификации по HTTP в качестве пароля вводится Personal Access Token, а не пароль учётной записи.
5. Особый случай: если репозиторий в GitLab был создан с инициализацией (README), удалённая ветка содержит коммиты, отсутствующие локально, и push отклоняется (`fetch first`). Надёжное разрешение — влить удалённую историю и опубликовать результат:
   ```bash
   git fetch origin
   git log --oneline origin/main
   git pull origin main --rebase --allow-unrelated-histories
   git push -u origin main
   ```
   Флаг `--allow-unrelated-histories` обязателен, поскольку локальная и удалённая истории созданы независимо.
6. Критерий завершения: в веб-интерфейсе GitLab видны все пять файлов проекта; команда `git status` сообщает `up to date with 'origin/main'`.

## 4. Интеграция GitLab и AWX

### 4.1. Deploy token (со стороны GitLab)

1. В проекте GitLab: *Settings → Repository → Deploy tokens*.
2. Создать токен с scope `read_repository`; значение отображается однократно и подлежит немедленному сохранению.

### 4.2. Credentials (со стороны AWX)

1. Source Control credential: *Resources → Credentials → Add*; тип Source Control; имя пользователя — имя deploy token, секрет — значение токена.
2. Machine credential: тип Machine; пользователь `test`; поле SSH Private Key заполняется полным текстом ключа (включая строки BEGIN/END), а не путём к файлу; Privilege Escalation Username — `root`.

Примечание: пароль привилегированного повышения не требуется, поскольку на целевом сервере настроен passwordless sudo.

### 4.3. Project

1. *Resources → Projects → Add*; поле Organization заполняется значением существующей организации (Default) из выпадающего списка; произвольный ввод не допускается.
2. SCM Type — Git; SCM URL — `http://192.168.0.111/test1/ansible-postgresql.git`; SCM Credential — credential из п. 4.2.1; ветка — `main`; опция Update Revision on Launch — включена.
3. Критерий готовности: статус проекта Successful; в перечне обнаруженных playbook присутствует `install_postgres.yml`.

### 4.4. Inventory

1. *Resources → Inventories → Add*: имя `db-inventory`, Organization — Default; форма создания содержит лишь базовые поля и сохраняется как есть.
2. Вкладка **Sources** доступна только после сохранения инвентаря — это штатное поведение AWX, а не дефект интерфейса. Внутри созданного инвентаря: *Sources → Add*; Source — Sourced from a Project; Project — `ansible-postgresql`; Inventory File — `inventory/hosts.yml` (без ведущего слэша); опция Overwrite — включена; сохранение и последующий запуск Sync.
3. Критерий готовности: источник Successful; на вкладке Hosts появился узел `awx_runner`.

### 4.5. Execution Environment (важная особенность стенда)

Выявленный на стенде дефект: дефолтный EE объявляет образ `quay.io/ansible/awx-ee:latest` с политикой pull Always, что при недоступности quay.io приводит к бессрочному зависанию подов заданий в состоянии ContainerCreating, даже если образ версии 24.6.1 присутствует в локальном кэше containerd. Устранение:

1. Присвоение кэшированному образу требуемого тега:
   ```bash
   sudo k3s ctr images tag quay.m.daocloud.io/ansible/awx-ee:24.6.1 quay.io/ansible/awx-ee:latest
   ```
2. Смена политики pull: *Administration → Execution Environments* → дефолтный EE → Pull — Never (либо Before update).
3. Принудительная очистка зависших подов заданий:
   ```bash
   kubectl delete pod -n awx <имя-пода> --grace-period=0 --force
   ```
   после чего задание перезапускается в AWX, а новый под создаётся мгновенно из локального образа.

Примечание: для изолированных контуров, где реестр quay.io недоступен, политика Never (или Before update) является корректной постоянной конфигурацией; политика Always в подобных сетях недопустима.

## 5. Настройка выполнения сценария в AWX

### 5.1. Job Template

1. *Resources → Templates → Add → Job Template*:
   - Name: `Install PostgreSQL`;
   - Project: `ansible-postgresql`; Playbook: `install_postgres.yml`;
   - Inventory: `db-inventory`;
   - Credentials: Machine credential из п. 4.2.2;
   - Enable Privilege Escalation — включено;
   - Organization — Default.
2. Сохранение шаблона обязательно до настройки survey, поскольку форма настроек шаблона и форма survey в AWX разделены.

### 5.2. Survey (входные данные задания)

Survey настраивается в шаблоне и содержит два вопроса:

- переменная `db_server_ip` — тип Text, ответ обязателен; назначение — IP-адрес сервера СУБД;
- переменная `pg_version` — тип Multiple Choice (single select), ответ обязателен; варианты `15`, `16`, `17`, `18`; значение по умолчанию — `17`.

Критически важное требование, подтверждённое практикой стенда: после добавления вопросов переключатель **Survey Enabled** должен быть установлен в положение ON, после чего форма сохраняется. При выключенном survey AWX не задаёт вопросов и не формирует переменные; задание, запущенное в таком состоянии, завершается ошибкой `'db_server_ip' is undefined` в задаче `add_host`, при этом сам плейбук и подключение к хосту остаются корректными. При запуске активного survey AWX в обязательном порядке отображает форму опроса — её отсутствие является диагностическим признаком выключенного survey.

## 6. Выполнение и контроль

1. Запуск: *Templates → Install PostgreSQL → Launch*; в форме survey указать IP сервера СУБД (например, `192.168.0.105`) и мажорную версию PostgreSQL (`17`).
2. Контрольные точки журнала задания:
   - задачи установки (`get_url`, `apt_repository`, `apt`) возвращают `changed`;
   - задачи `lineinfile` возвращают `changed`;
   - финальная проверка `pg_isready` возвращает `accepting connections`;
   - сводка (Recap) зелёная.
3. Контроль на сервере СУБД:
   ```bash
   systemctl status postgresql@17-main
   ss -tlnp | grep 5432
   sudo -u postgres psql -c "SELECT version();"
   ```
4. Проверка идемпотентности: повторный запуск шаблона с теми же параметрами должен показать все задачи в состоянии `ok` и отсутствие `changed`. Присутствие `changed` при повторном прогоне свидетельствует о неидемпотентной задаче и подлежит устранению.

## 7. Сводка выявленных типовых препятствий и способов их устранения

- Ошибка `src refspec main does not match any` — расхождение имён локальной и запрашиваемой веток; устраняется командой `git branch -m master main`.
- Отклонение push (`fetch first`) — наличие удалённой истории, отсутствующей локально (инициализация репозитория с README); устраняется `git pull --rebase --allow-unrelated-histories` либо, при заведомо пустом содержимом, `push --force`.
- Аутентификация по HTTP в GitLab — требуется Personal Access Token, а не пароль учётной записи.
- Отсутствие вкладки Sources у нового инвентаря — штатное поведение; источники добавляются после сохранения инвентаря.
- `AxiosError: Network Error` в интерфейсе AWX — сбой доступности API (истёкшая сессия или перезапуск подов после перезагрузки виртуальной машины); устраняется перелогином и контролем состояния подов (`kubectl get pods -n awx`).
- Бессрочное `ContainerCreating` подов заданий — образ EE объявлен тегом `latest` с политикой pull Always при недоступности quay.io; устраняется тегированием локального образа и установкой политики Pull = Never (Before update).
- Ошибка `'db_server_ip' is undefined` при запуске задания — survey не активирован (переключатель Survey Enabled выключен) либо переменные не переданы; устраняется активацией survey и проверкой Extra Variables в деталях задания.
- Предупреждение о `ANSIBLE_COLLECTIONS_PATH` в журнале — устаревшая переменная окружения внутри EE; на работоспособность не влияет и устранения не требует.

## 8. Рекомендации по дальнейшему развитию решения

- Защита ветки `main` в GitLab (*Settings → Repository → Protected branches*) после перехода проекта в статус рабочего источника для AWX.
- Кэширование учётных данных Git (`git config --global credential.helper store`) для сокращения повторных вводов токена.
- Мониторинг ресурсопотребления узла k3s: перезапуск виртуальной машины, наблюдавшийся в ходе работ, следует отличать от аварийного завершения подов AWX по исчерпанию памяти (проверка `journalctl -k -b -1 | grep -i oom`); при подтверждении OOM требуется увеличение лимитов ресурсов подов или объёма ВМ.

Результатом выполнения настоящей методики является полностью автоматизированный контур: установка PostgreSQL заданной мажорной версии на произвольный сервер СУБД выполняется оператором в AWX вводом двух параметров (IP-адрес и версия), при этом исходный код автоматизации версионируется в GitLab и поддерживает повторяемость и идемпотентность исполнения.
