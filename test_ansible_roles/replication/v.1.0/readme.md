Настройка репликации — это логичное продолжение ролей установки и обновления. Здесь ключевая сложность в том, что **роль должна по-разному вести себя на primary и standby узлах**, сохраняя при этом единую точку управления. Ниже — структура, которая решает эту задачу через одну роль с условным выполнением задач в зависимости от роли сервера.

### 🧭 Принципы построения роли репликации

- **Единая роль для primary и replica**: Вместо двух отдельных ролей используется одна, а поведение определяется переменной `postgresql_role` (значения `primary` или `replica`) для каждого хоста в инвентаре. Это гарантирует согласованность настроек и упрощает поддержку.
- **Идемпотентная инициализация реплик**: Задача создания standby не должна пытаться заново скопировать данные, если реплика уже настроена. Проверка наличия файла `standby.signal` или запись в `pg_stat_replication` помогает избежать повторного выполнения `pg_basebackup`.
- **Слоты репликации для надежности**: Использование физических слотов репликации (по одному на каждую реплику) предотвращает удаление WAL-сегментов, которые еще нужны standby-узлам, даже если primary испытывает нагрузку.
- **Разделение ответственности**: Задачи внутри роли разделены на блоки, которые выполняются только на primary, только на replica, или на обоих типах узлов.

### 📂 Структура каталогов и файлов роли

```text
roles/postgresql-replication/
├── defaults/
│   └── main.yml              # Переменные по умолчанию (имена пользователей, параметры WAL)
├── tasks/
│   ├── main.yml              # Оркестратор: вызов общих задач и задач по ролям
│   ├── common.yml            # Установка пакетов, базовые настройки (для всех узлов)
│   ├── primary.yml           # Настройка primary: pg_hba, пользователь, слоты
│   ├── replica.yml           # Настройка standby: pg_basebackup, параметры подключения
│   └── verify.yml            # Проверка статуса репликации (опционально)
├── handlers/
│   └── main.yml              # Обработчики (reload/restart PostgreSQL)
├── templates/
│   ├── postgresql.conf.j2    # Основной конфиг (параметры WAL, репликации)
│   ├── pg_hba.conf.j2        # Правила доступа (разрешение для replicator)
│   └── primary_conninfo.j2   # (опционально) Шаблон для настройки подключения
└── vars/
    └── main.yml              # Внутренние переменные (пути, вычисляемые значения)
```

### 📝 Ключевые файлы и их назначение

**`defaults/main.yml`** — определяет параметры, которые могут различаться для разных кластеров.

```yaml
# Роль узла (переопределяется в инвентаре для каждого хоста)
postgresql_role: "replica"  # "primary" или "replica"

# Учетная запись для репликации
postgresql_replication_user: "replicator"
postgresql_replication_password: "{{ vault_replication_password }}"

# Параметры WAL и репликации (для primary)
postgresql_max_wal_senders: 10
postgresql_wal_level: "replica"  # или "logical" при необходимости
postgresql_wal_keep_size: "2GB"

# Список реплик (используется для создания слотов и записей в pg_hba)
postgresql_replicas: "{{ groups['db_replicas'] | default([]) }}"

# Параметры для standby
postgresql_primary_host: "{{ hostvars[groups['db_primary'][0]]['ansible_host'] }}"
postgresql_primary_port: 5432

# Директории (пример для Debian/Ubuntu)
postgresql_config_dir: "/etc/postgresql/{{ postgresql_version }}/main"
postgresql_data_dir: "/var/lib/postgresql/{{ postgresql_version }}/main"
```

**`tasks/main.yml`** — главный файл, который применяет общие задачи, а затем вызывает задачи для конкретной роли узла.

```yaml
---
- name: Apply common configuration
  ansible.builtin.include_tasks: common.yml
  tags: common

- name: Configure primary node
  ansible.builtin.include_tasks: primary.yml
  when: postgresql_role == "primary"
  tags: primary

- name: Configure replica node
  ansible.builtin.include_tasks: replica.yml
  when: postgresql_role == "replica"
  tags: replica

- name: Verify replication status
  ansible.builtin.include_tasks: verify.yml
  tags: verify
  when: postgresql_role == "primary" or postgresql_role == "replica"
```
Тег `verify` позволяет запускать проверку отдельно (`--tags verify`), не выполняя всю настройку.

**`tasks/primary.yml`** — настройка primary: разрешения в `pg_hba.conf`, создание пользователя и слотов репликации.

```yaml
---
- name: Ensure PostgreSQL is running
  ansible.builtin.service:
    name: postgresql
    state: started
    enabled: yes

- name: Create replication user
  community.postgresql.postgresql_user:
    name: "{{ postgresql_replication_user }}"
    password: "{{ postgresql_replication_password }}"
    role_attr_flags: "REPLICATION,LOGIN"
    state: present
  become_user: postgres
  no_log: true

- name: Create physical replication slots for each replica
  community.postgresql.postgresql_slot:
    name: "{{ hostvars[item].inventory_hostname | replace('.', '_') | replace('-', '_') }}"
    slot_type: physical
    state: present
  become_user: postgres
  loop: "{{ postgresql_replicas }}"
  when: postgresql_replicas | length > 0
```
Слоты именуются на основе имени хоста, чтобы каждый standby имел собственный слот.

**`tasks/replica.yml`** — инициализация standby. Ключевой момент — **не выполнять `pg_basebackup` повторно**, если реплика уже настроена.

```yaml
---
- name: Check if replica is already initialized
  ansible.builtin.stat:
    path: "{{ postgresql_data_dir }}/standby.signal"
  register: standby_signal

- name: Stop PostgreSQL on replica before base backup
  ansible.builtin.service:
    name: postgresql
    state: stopped
  when: not standby_signal.stat.exists

- name: Run pg_basebackup to seed the replica
  ansible.builtin.command:
    cmd: >
      pg_basebackup
      -h {{ postgresql_primary_host }}
      -p {{ postgresql_primary_port }}
      -U {{ postgresql_replication_user }}
      -D {{ postgresql_data_dir }}
      -Fp -Xs -P -R
  become_user: postgres
  when: not standby_signal.stat.exists
  # Флаг -R автоматически создает standby.signal и прописывает primary_conninfo

- name: Ensure PostgreSQL is running on replica
  ansible.builtin.service:
    name: postgresql
    state: started
    enabled: yes
```
Проверка `standby.signal` делает задачу идемпотентной: при повторном запуске роли `pg_basebackup` не будет выполняться, если реплика уже развернута.

**`tasks/verify.yml`** — опциональная проверка, что репликация действительно работает.

```yaml
---
- name: Query replication status on primary
  community.postgresql.postgresql_query:
    query: "SELECT client_addr, state, sync_state FROM pg_stat_replication;"
  become_user: postgres
  register: repl_status
  when: postgresql_role == "primary"

- name: Show replication status
  ansible.builtin.debug:
    var: repl_status.query_result
  when: postgresql_role == "primary"
```
Этот шаг помогает убедиться, что standby-узлы подключились и находятся в состоянии `streaming`.

### 🖥️ Инвентарь и плейбук

Роль опирается на группы `db_primary` и `db_replicas`. Переменная `postgresql_role` задается для каждого хоста.

**`inventories/production/hosts.yml`:**
```yaml
all:
  children:
    db_primary:
      hosts:
        pg-primary.example.com:
          ansible_host: 10.0.2.10
          postgresql_role: primary
    db_replicas:
      hosts:
        pg-replica01.example.com:
          ansible_host: 10.0.2.11
          postgresql_role: replica
        pg-replica02.example.com:
          ansible_host: 10.0.2.12
          postgresql_role: replica
    dbservers:
      children:
        db_primary:
        db_replicas:
```

**`group_vars/dbservers.yml`** — общие переменные для всех узлов:
```yaml
postgresql_version: 16
postgresql_replication_user: "replicator"
postgresql_replication_password: "{{ vault_replication_password }}"
```

**`playbook-replication.yml`:**
```yaml
- name: Configure PostgreSQL streaming replication
  hosts: dbservers
  become: yes
  roles:
    - postgresql-replication
```

### 💡 Дополнительные рекомендации

- **Синхронная репликация**: Если требуется гарантия отсутствия потери данных, настройте `synchronous_standby_names` на primary. Это можно сделать через шаблон `postgresql.conf.j2` или модуль `community.postgresql.postgresql_set` (хотя он помечен как deprecated, для точечных изменений можно использовать `postgresql_query` с `ALTER SYSTEM`).
- **Автоматический failover**: Роль настраивает базовую потоковую репликацию, но не управляет повышением standby до primary. Для production-средств стоит рассмотреть интеграцию с **repmgr** или **Patroni**. Это можно вынести в отдельную роль или слой поверх текущей.
- **Мониторинг**: Добавьте задачу, которая проверяет задержку репликации (`pg_stat_replication.replay_lag`) и отправляет метрики в систему мониторинга. Это логично вынести в отдельный playbook или роль.
- **Безопасность**: Всегда используйте `no_log: true` для задач, работающих с паролями, и храните `vault_replication_password` в Ansible Vault.

В итоге структура повторяет подход из предыдущих ролей: единая точка входа, четкое разделение этапов по тегам и идемпотентность за счет проверки состояния перед деструктивными действиями.
