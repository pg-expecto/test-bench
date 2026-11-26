Вот общая структура Ansible-роли для установки и настройки PostgreSQL на группе серверов. Такая структура позволяет централизованно управлять конфигурацией для всего списка СУБД.

### 📂 Структура каталогов и файлов роли

Стандартный скелет роли создается командой `ansible-galaxy role init --init-path roles/ postgresql` и выглядит так:

```text
roles/postgresql/
├── defaults/
│   └── main.yml          # Переменные по умолчанию (с низким приоритетом)
├── files/
│   └── (опционально)     # Статические файлы для копирования (например, скрипты)
├── handlers/
│   └── main.yml          # Обработчики (например, перезапуск PostgreSQL)
├── meta/
│   └── main.yml          # Метаданные роли и зависимости
├── tasks/
│   ├── main.yml          # Главный файл, оркестрирующий задачи
│   ├── install.yml       # Установка пакетов и репозиториев
│   ├── configure.yml     # Настройка конфигурации
│   └── users_and_dbs.yml # Создание пользователей и баз данных
├── templates/
│   ├── postgresql.conf.j2 # Шаблон основного конфигурационного файла
│   └── pg_hba.conf.j2     # Шаблон файла аутентификации клиентов
├── tests/
│   └── (опционально)     # Тесты для роли (например, Molecule)
└── vars/
    └── main.yml          # Переменные с высоким приоритетом (используются редко)
```

### 📝 Ключевые файлы и их назначение

**`defaults/main.yml`** — здесь определяются все настраиваемые параметры с разумными значениями по умолчанию. Пример для управления списком серверов и версией СУБД:

```yaml
# Версия PostgreSQL и пакеты
pg_version: 16
pg_packages:
  - "postgresql-{{ pg_version }}"
  - "postgresql-client-{{ pg_version }}"
  - "postgresql-contrib-{{ pg_version }}"
  - python3-psycopg2  # Драйвер Python для модулей Ansible

# Основные директории (зависят от ОС, пример для Debian/Ubuntu)
pg_config_dir: "/etc/postgresql/{{ pg_version }}/main"
pg_data_dir: "/var/lib/postgresql/{{ pg_version }}/main"

# Параметры подключения
pg_listen_addresses: "localhost"  # Для доступа извне укажите "*" или конкретный IP
pg_port: 5432

# Параметры производительности (адаптируйте под RAM серверов)
pg_shared_buffers: "256MB"
pg_effective_cache_size: "768MB"

# Список баз данных и пользователей для создания
pg_databases: []
pg_users: []
```


**`tasks/main.yml`** — главный файл, который подключает остальные задачи. Это позволяет легко включать или отключать определенные этапы с помощью тегов.

```yaml
---
- name: Include installation tasks
  ansible.builtin.include_tasks: install.yml
  tags: install

- name: Include configuration tasks
  ansible.builtin.include_tasks: configure.yml
  tags: configure

- name: Include users and databases management
  ansible.builtin.include_tasks: users_and_dbs.yml
  tags: users_dbs
```

**`tasks/install.yml`** — задачи по добавлению официального репозитория PGDG и установке пакетов.

**`tasks/configure.yml`** — развертывание конфигурационных файлов из шаблонов и уведомление обработчика о необходимости перезапуска службы при изменениях.

**`templates/postgresql.conf.j2` и `templates/pg_hba.conf.j2`** — Jinja2-шаблоны, которые используют переменные из `defaults/main.yml` для генерации итоговых конфигураций на каждом сервере.

**`handlers/main.yml`** — определяет действие, которое вызывается задачами при изменении конфигурации (обычно перезапуск службы).

### 🖥️ Применение к списку серверов (Inventory)

Чтобы роль применилась ко всем серверам СУБД, они должны быть объединены в группу инвентаря. Переменные, специфичные для этой группы (например, версия PostgreSQL или параметры памяти), удобно хранить в `group_vars`.

**Пример инвентаря (`inventories/production/hosts.yml`):**
```yaml
all:
  children:
    dbservers:
      hosts:
        pg-node-01:
          ansible_host: 192.168.1.10
        pg-node-02:
          ansible_host: 192.168.1.11
        pg-node-03:
          ansible_host: 192.168.1.12
```

**Пример групповых переменных (`inventories/production/group_vars/dbservers.yml`):**
```yaml
# Переопределяем переменные роли для всех серверов группы
pg_version: 15
pg_shared_buffers: "1GB"  # Больше памяти для продакшн-серверов
pg_listen_addresses: "*"
```

**Плейбук для запуска (`playbook.yml`):**
```yaml
- name: Установка и настройка PostgreSQL
  hosts: dbservers  # Роль применится ко всем хостам в этой группе
  become: yes
  roles:
    - postgresql
```
Запуск выполняется командой `ansible-playbook -i inventories/production/hosts.yml playbook.yml`.

### 💡 Дополнительные рекомендации

- **Чувствительные данные**: Пароли пользователей БД не стоит хранить в открытом виде в `defaults` или `group_vars`. Используйте **Ansible Vault** для шифрования файлов с секретами.
- **Коллекции**: Для управления пользователями, базами данных и расширениями потребуется коллекция `community.postgresql`. Убедитесь, что она установлена на control node.
- **Идемпотентность**: Все задачи должны быть идемпотентными, чтобы повторный запуск плейбука не приводил к ошибкам или нежелательным изменениям.
