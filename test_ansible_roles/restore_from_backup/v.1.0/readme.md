В отличие от ролей установки или репликации, роль восстановления — это **не декларативное приведение к состоянию, а управляемая деструктивная операция**. Её главная задача — гарантировать, что восстановление произойдёт предсказуемо, безопасно и только тогда, когда это действительно нужно, с обязательной проверкой целостности данных после завершения.

### 🧭 Принципы построения роли восстановления

- **Восстановление как отдельная операция, а не часть деплоя**: Роль не должна запускаться при каждом обычном прогоне плейбука. Она вызывается либо вручную для disaster recovery, либо по расписанию для регулярной проверки бэкапов на тестовых стендах.
- **Идемпотентность через маркер или проверку**: Повторный запуск роли не должен приводить к повторному восстановлению той же самой копии. Для этого используется либо файл-маркер с идентификатором бэкапа, либо проверка, что целевая база уже содержит данные из этой копии.
- **Строгая последовательность**: Скачивание и проверка целостности → остановка сервисов/приложений → подготовка чистой БД → восстановление → пост-обработка (`ANALYZE`) → проверка → запуск сервисов.
- **Защита от случайного запуска**: Деструктивные задачи (например, `DROP DATABASE`) должны быть защищены флагом подтверждения (аналогично `pg_upgrade_confirm` из роли обновления), чтобы предотвратить случайное выполнение на продакшене.

### 📂 Структура каталогов и файлов роли

```text
roles/postgresql-restore/
├── defaults/
│   └── main.yml              # Параметры по умолчанию (источник бэкапа, имена БД, флаги)
├── tasks/
│   ├── main.yml              # Оркестратор этапов с тегами
│   ├── download.yml          # Получение бэкапа из хранилища и проверка целостности
│   ├── pre_restore.yml       # Остановка сервисов, завершение сессий, подготовка БД
│   ├── restore.yml           # Непосредственно восстановление (pg_restore/psql)
│   ├── post_restore.yml      # ANALYZE, обновление расширений, сброс маркера
│   └── verify.yml            # Проверка, что данные восстановлены и БД доступна
├── handlers/
│   └── main.yml              # Обработчики (запуск/остановка PostgreSQL, приложений)
└── vars/
    └── main.yml              # Внутренние вычисляемые переменные (пути, имена файлов)
```

### 📝 Ключевые файлы и их назначение

**`defaults/main.yml`** — определяет, *что* и *куда* восстанавливать.

```yaml
# Флаг подтверждения: без него роль не выполнит деструктивные шаги
postgresql_restore_confirm: false

# Источник бэкапа
postgresql_backup_source: "s3"          # "s3", "nfs", "local"
postgresql_backup_bucket: "mycompany-db-backups"
postgresql_backup_date: "{{ lookup('pipe', 'date +%Y-%m-%d') }}"  # по умолчанию — сегодня
postgresql_backup_file: "postgresql_{{ postgresql_backup_date }}.dump"
postgresql_restore_dir: "/var/backups/restore"

# Параметры целевой БД (если восстанавливается одна база из дампа)
postgresql_restore_db_name: "myapp_production"
postgresql_restore_db_owner: "myapp"

# Административные учетные данные
postgresql_admin_user: "postgres"
postgresql_admin_password: "{{ vault_pg_admin_password }}"

# Параметры pg_restore
postgresql_restore_jobs: 4
postgresql_restore_no_owner: true
```
Флаг `postgresql_restore_confirm` (по аналогии с ролью обновления) гарантирует, что роль не начнёт удалять базы данных без явного подтверждения оператора.

**`tasks/main.yml`** — оркестратор, который позволяет запускать этапы отдельно.

```yaml
---
- name: Download and verify backup integrity
  ansible.builtin.include_tasks: download.yml
  tags: [download, restore-check]

- name: Prepare target for restore (stop services, clean database)
  ansible.builtin.include_tasks: pre_restore.yml
  tags: [pre-restore]

- name: Perform the actual restore
  ansible.builtin.include_tasks: restore.yml
  tags: [restore]

- name: Post-restore maintenance (ANALYZE, extensions)
  ansible.builtin.include_tasks: post_restore.yml
  tags: [post-restore]

- name: Verify restored data
  ansible.builtin.include_tasks: verify.yml
  tags: [verify]
```
Такая разбивка позволяет, например, выполнить только `--tags download,restore-check`, чтобы убедиться, что бэкап скачан и валиден, прежде чем запускать деструктивные шаги.

**`tasks/download.yml`** — скачивание и проверка целостности. Пример для S3, но может быть адаптирован под NFS или локальный путь.

```yaml
---
- name: Ensure restore directory exists
  ansible.builtin.file:
    path: "{{ postgresql_restore_dir }}"
    state: directory
    mode: "0750"

- name: Download backup file from S3
  ansible.builtin.command:
    cmd: >
      aws s3 cp
      s3://{{ postgresql_backup_bucket }}/{{ postgresql_backup_file }}
      {{ postgresql_restore_dir }}/{{ postgresql_backup_file }}
  environment:
    AWS_ACCESS_KEY_ID: "{{ vault_aws_access_key }}"
    AWS_SECRET_ACCESS_KEY: "{{ vault_aws_secret_key }}"
  register: download_result
  changed_when: download_result.rc == 0
  no_log: true

- name: Verify backup file is not empty
  ansible.builtin.stat:
    path: "{{ postgresql_restore_dir }}/{{ postgresql_backup_file }}"
  register: backup_stat

- name: Fail if backup file is missing or too small
  ansible.builtin.fail:
    msg: "Backup file is missing or suspiciously small ({{ backup_stat.stat.size | default(0) }} bytes)"
  when: not backup_stat.stat.exists or backup_stat.stat.size < 1024
```
Проверка размера файла (как в примере OneUptime) отсеивает явно битые или пустые загрузки.

**`tasks/pre_restore.yml`** — остановка приложений и подготовка чистой БД.

```yaml
---
- name: Ensure destructive operations are explicitly confirmed
  ansible.builtin.fail:
    msg: "Set postgresql_restore_confirm=true to proceed with restore"
  when: not postgresql_restore_confirm

- name: Stop application services that use the database
  ansible.builtin.service:
    name: "{{ item }}"
    state: stopped
  loop: "{{ postgresql_dependent_services | default([]) }}"

- name: Terminate all active connections to the target database
  community.postgresql.postgresql_query:
    db: postgres
    login_user: "{{ postgresql_admin_user }}"
    login_password: "{{ postgresql_admin_password }}"
    query: >
      SELECT pg_terminate_backend(pid)
      FROM pg_stat_activity
      WHERE datname = %s AND pid != pg_backend_pid();
    positional_args:
      - "{{ postgresql_restore_db_name }}"
  no_log: true

- name: Drop the existing target database (if it exists)
  community.postgresql.postgresql_db:
    name: "{{ postgresql_restore_db_name }}"
    login_user: "{{ postgresql_admin_user }}"
    login_password: "{{ postgresql_admin_password }}"
    state: absent
  no_log: true

- name: Create a fresh empty database owned by the application role
  community.postgresql.postgresql_db:
    name: "{{ postgresql_restore_db_name }}"
    owner: "{{ postgresql_restore_db_owner }}"
    login_user: "{{ postgresql_admin_user }}"
    login_password: "{{ postgresql_admin_password }}"
    state: present
  no_log: true
```
Этот шаг повторяет логику из практик восстановления: завершение сессий → удаление старой БД → создание чистой БД.

**`tasks/restore.yml`** — само восстановление. Для дампов в формате `custom`/`directory` используется `pg_restore`, для plain-text (`pg_dumpall`) — `psql`.

```yaml
---
- name: Restore database from custom-format dump using pg_restore
  ansible.builtin.command:
    cmd: >
      pg_restore
      --host=localhost
      --username={{ postgresql_admin_user }}
      --dbname={{ postgresql_restore_db_name }}
      --jobs={{ postgresql_restore_jobs }}
      {% if postgresql_restore_no_owner %}--no-owner{% endif %}
      --role={{ postgresql_restore_db_owner }}
      --verbose
      {{ postgresql_restore_dir }}/{{ postgresql_backup_file }}
  environment:
    PGPASSWORD: "{{ postgresql_admin_password }}"
  register: restore_result
  changed_when: restore_result.rc == 0
  no_log: true
  when: postgresql_backup_file is search('\\.dump$|\\.backup$')

- name: Restore from plain-text SQL dump using psql
  ansible.builtin.shell:
    cmd: >
      psql -h localhost -U {{ postgresql_admin_user }}
      -d {{ postgresql_restore_db_name }}
      -f {{ postgresql_restore_dir }}/{{ postgresql_backup_file }}
  environment:
    PGPASSWORD: "{{ postgresql_admin_password }}"
  register: psql_restore_result
  changed_when: psql_restore_result.rc == 0
  no_log: true
  when: postgresql_backup_file is search('\\.sql$')
```
Выбор команды зависит от формата бэкапа: `pg_restore` для archive-форматов, `psql` для текстовых дампов.

**`tasks/post_restore.yml`** — обязательный `ANALYZE` и обновление расширений.

```yaml
---
- name: Run ANALYZE on the restored database to refresh planner statistics
  ansible.builtin.command:
    cmd: >
      psql -h localhost -U {{ postgresql_admin_user }}
      -d {{ postgresql_restore_db_name }}
      -c "ANALYZE VERBOSE"
  environment:
    PGPASSWORD: "{{ postgresql_admin_password }}"
  changed_when: true
  no_log: true

- name: Update PostgreSQL extensions in the restored database
  community.postgresql.postgresql_query:
    db: "{{ postgresql_restore_db_name }}"
    login_user: "{{ postgresql_admin_user }}"
    login_password: "{{ postgresql_admin_password }}"
    query: "ALTER EXTENSION {{ item }} UPDATE;"
  loop: "{{ postgresql_extensions_to_update | default([]) }}"
  no_log: true
```
`ANALYZE` после восстановления — стандартная практика, без неё планировщик может принимать неоптимальные решения.

**`tasks/verify.yml`** — проверка, что база доступна и содержит данные.

```yaml
---
- name: Query row count in a key table to verify data presence
  community.postgresql.postgresql_query:
    db: "{{ postgresql_restore_db_name }}"
    login_user: "{{ postgresql_admin_user }}"
    login_password: "{{ postgresql_admin_password }}"
    query: "SELECT count(*) AS row_count FROM {{ postgresql_verify_table | default('information_schema.tables') }};"
  register: verify_result
  changed_when: false
  no_log: true

- name: Fail if verification query returns no rows
  ansible.builtin.fail:
    msg: "Post-restore verification failed: no rows found in {{ postgresql_verify_table | default('information_schema.tables') }}"
  when: verify_result.query_result[0].row_count | int == 0

- name: Start application services back
  ansible.builtin.service:
    name: "{{ item }}"
    state: started
  loop: "{{ postgresql_dependent_services | default([]) }}"
```
Проверка наличия строк в ключевой таблице (или в системном каталоге) даёт базовую уверенность, что восстановление не завершилось «пустой» базой.

### 🖥️ Инвентарь и плейбук

Роль применяется к группе хостов, предназначенных для восстановления (тестовые стенды, резервные серверы или сам primary в случае DR). Это отделяет её от группы рабочих `dbservers`.

**`inventories/restore/hosts.yml`:**
```yaml
all:
  children:
    restore_targets:
      hosts:
        pg-restore-test01:
          ansible_host: 10.0.3.10
        pg-restore-test02:
          ansible_host: 10.0.3.11
```

**`group_vars/restore_targets.yml`:**
```yaml
postgresql_restore_confirm: true
postgresql_backup_source: "s3"
postgresql_restore_db_name: "myapp_production"
postgresql_restore_db_owner: "myapp"
postgresql_dependent_services:
  - myapp-backend
  - myapp-worker
postgresql_verify_table: "public.users"
```

**`playbook-restore.yml`:**
```yaml
- name: Restore PostgreSQL from full backup
  hosts: restore_targets
  become: yes
  serial: 1  # при необходимости восстанавливаем серверы по очереди
  roles:
    - postgresql-restore
```
Запуск только проверки бэкапа: `ansible-playbook ... --tags download,restore-check`. Полное восстановление с подтверждением: `ansible-playbook ... -e postgresql_restore_confirm=true`.

### 💡 Дополнительные рекомендации

- **Регулярное тестирование восстановления**: Роль удобно запускать по расписанию (через cron или AWX/Tower) на тестовом стенде с актуальным бэкапом. Это единственный способ убедиться, что резервные копии действительно пригодны для восстановления. Как отмечается в руководствах, «процедура восстановления, которая никогда не тестировалась, — это liability».
- **Полное восстановление кластера (`pg_dumpall`)**: Если бэкап представляет собой дамп всего кластера (все базы и роли), шаги `pre_restore` и `restore` должны быть изменены: вместо удаления одной БД выполняется остановка PostgreSQL, очистка `data directory` и восстановление через `psql` от имени суперпользователя. Такая логика ближе к роли мажорного обновления и должна быть выделена в отдельный набор задач (например, `restore_cluster.yml`), включаемый по условию `postgresql_restore_scope: "cluster"`.
- **Маркер выполнения**: Для истинной идемпотентности в регулярном сценарии можно записывать идентификатор восстановленного бэкапа в файл (например, `{{ postgresql_data_dir }}/.last_restored_backup`) и пропускать восстановление, если он совпадает с текущим `postgresql_backup_file`. Это предотвратит повторное удаление базы при случайном повторном запуске.
- **Безопасность**: Все задачи, работающие с паролями, должны иметь `no_log: true`. Учетные данные для S3 и PostgreSQL хранятся в Ansible Vault.

В итоге структура роли для восстановления следует тому же принципу, что и предыдущие роли: **чёткие этапы с тегами, защита от случайных деструктивных действий и обязательная пост-проверка**, но с акцентом на то, что это операция, которая должна выполняться осознанно и контролируемо.
