Мажорное обновление PostgreSQL (например, с 15 на 16) — принципиально иная задача, чем первичная установка. Внутренний формат данных меняется, поэтому просто обновить пакеты и перезапустить службу нельзя. Роль должна быть построена как **поэтапный процесс с обязательными точками проверки и возможностью отката**.

### 🧭 Ключевые принципы построения роли

В отличие от роли установки, здесь нельзя полагаться на простую идемпотентность. Основной упор делается на **управление состоянием** и **безопасность**:

- **Предварительные проверки (`--check`)**: Перед любыми изменениями необходимо убедиться, что `pg_upgrade --check` не выявит несовместимостей схемы или кластера.
- **Резервное копирование**: Обязательный шаг. Даже при использовании `pg_upgrade` с жесткими ссылками (`--link`) рекомендуется иметь логический дамп (`pg_dumpall`) на случай критического сбоя.
- **Стратегия миграции**: Роль должна поддерживать как минимум два режима: быстрый `pg_upgrade` (in-place) и более медленный, но иногда более предсказуемый `dump/restore`.
- **Пошаговое выполнение**: Задачи не должны пытаться обновить весь кластер за один прогон. Лучше разделить процесс на этапы, которые можно запускать с тегами (например, `pre-checks`, `migrate`, `switchover`).

### 📂 Структура каталогов и файлов роли

Рекомендуется создать отдельную роль, например, `postgresql-upgrade`, чтобы не смешивать логику установки и обновления.

```text
roles/postgresql-upgrade/
├── defaults/
│   └── main.yml              # Переменные по умолчанию (версии, пути, режим)
├── tasks/
│   ├── main.yml              # Оркестратор: вызов этапов по тегам
│   ├── pre_checks.yml        # Проверка совместимости и готовности
│   ├── backup.yml            # Создание резервной копии (логической или физической)
│   ├── install_new_binaries.yml # Установка пакетов новой версии (без запуска)
│   ├── migrate.yml           # Непосредственно миграция (pg_upgrade или dump/restore)
│   ├── switchover.yml        # Переключение на новую версию и запуск
│   └── rollback.yml          # Откат на предыдущую версию при сбое
├── handlers/
│   └── main.yml              # Обработчики (например, остановка/запуск службы)
├── templates/
│   └── (опционально)         # Шаблоны для временных конфигов, если нужны
└── vars/
    └── main.yml              # Внутренние переменные (пути, вычисляемые значения)
```

### 📝 Ключевые файлы и их назначение

**`defaults/main.yml`** — здесь определяются параметры, которые будут различаться для разных обновлений.

```yaml
# Версии
pg_old_version: 15
pg_new_version: 16

# Режим обновления: "pg_upgrade" или "dump_restore"
pg_upgrade_mode: "pg_upgrade"

# Для pg_upgrade: метод переноса файлов.
# "link" — быстро, но старая версия становится непригодной для запуска.
# "copy" — безопаснее (старый кластер остается), но требует много диска.
pg_upgrade_transfer_mode: "link"

# Пути (пример для Debian/Ubuntu, зависят от версии и ОС)
pg_old_bindir: "/usr/lib/postgresql/{{ pg_old_version }}/bin"
pg_old_datadir: "/var/lib/postgresql/{{ pg_old_version }}/main"
pg_new_bindir: "/usr/lib/postgresql/{{ pg_new_version }}/bin"
pg_new_datadir: "/var/lib/postgresql/{{ pg_new_version }}/main"

# Флаг для принудительного выполнения деструктивных операций (защита от случайного запуска)
pg_upgrade_confirm: false
```

**`tasks/main.yml`** — главный файл, который использует теги для управления этапами. Это позволяет запускать отдельные части пайплайна.

```yaml
---
- name: Preliminary compatibility checks
  ansible.builtin.include_tasks: pre_checks.yml
  tags: [pre-checks, upgrade-check]

- name: Create backup before upgrade
  ansible.builtin.include_tasks: backup.yml
  tags: [backup]

- name: Install new version binaries (without starting service)
  ansible.builtin.include_tasks: install_new_binaries.yml
  tags: [install-new]

- name: Perform data migration
  ansible.builtin.include_tasks: migrate.yml
  tags: [migrate]

- name: Switchover to new version
  ansible.builtin.include_tasks: switchover.yml
  tags: [switchover]

- name: Rollback procedure (manual trigger)
  ansible.builtin.include_tasks: rollback.yml
  tags: [never, rollback]
```
Обратите внимание на тег `never` у задачи `rollback` — она не выполнится при обычном запуске, только если явно указать `--tags rollback`.

**`tasks/pre_checks.yml`** — критически важный этап. Он должен остановить плейбук, если обнаружены проблемы.

```yaml
---
- name: Ensure old version is stopped
  ansible.builtin.service:
    name: "postgresql@{{ pg_old_version }}-main"
    state: stopped
  # Остановка нужна для корректной работы pg_upgrade --check

- name: Run pg_upgrade compatibility check
  ansible.builtin.command:
    cmd: >
      {{ pg_new_bindir }}/pg_upgrade
      --old-bindir {{ pg_old_bindir }}
      --new-bindir {{ pg_new_bindir }}
      --old-datadir {{ pg_old_datadir }}
      --new-datadir {{ pg_new_datadir }}
      --check
  register: pg_upgrade_check
  changed_when: false
  failed_when: pg_upgrade_check.rc != 0

- name: Display check result
  ansible.builtin.debug:
    msg: "Compatibility check passed: {{ pg_upgrade_check.stdout }}"
```
Этот шаг соответствует рекомендациям из практик обновления.

**`tasks/migrate.yml`** — реализация выбранной стратегии. В примере показан вызов `pg_upgrade` с параметрами.

```yaml
---
- name: Run pg_upgrade ({{ pg_upgrade_transfer_mode }} mode)
  ansible.builtin.command:
    cmd: >
      {{ pg_new_bindir }}/pg_upgrade
      --old-bindir {{ pg_old_bindir }}
      --new-bindir {{ pg_new_bindir }}
      --old-datadir {{ pg_old_datadir }}
      --new-datadir {{ pg_new_datadir }}
      --{{ pg_upgrade_transfer_mode }}
      --username=postgres
  when: pg_upgrade_mode == "pg_upgrade"
  register: pg_upgrade_result

- name: Alternative: Dump and restore strategy
  block:
    - name: Dump all databases
      ansible.builtin.shell: >
        {{ pg_old_bindir }}/pg_dumpall -U postgres > /tmp/pg_upgrade_dump.sql
    - name: Restore into new cluster
      ansible.builtin.shell: >
        {{ pg_new_bindir }}/psql -U postgres -f /tmp/pg_upgrade_dump.sql
  when: pg_upgrade_mode == "dump_restore"
```

### 🖥️ Применение к списку серверов (Inventory)

Обновление обычно затрагивает всю группу `dbservers`. В инвентаре можно указать разные версии для разных сред, но для продакшена важна последовательность.

**`inventories/production/group_vars/dbservers.yml`:**
```yaml
pg_old_version: 15
pg_new_version: 16
pg_upgrade_mode: "pg_upgrade"
pg_upgrade_transfer_mode: "copy"  # Безопаснее для критичных данных
```
Для поэтапного обновления (сначала реплики, потом мастер) можно использовать `serial` в плейбуке.

**`playbook-upgrade.yml`:**
```yaml
- name: Major PostgreSQL upgrade
  hosts: dbservers
  become: yes
  serial: 1  # Обновляем по одному серверу за раз
  roles:
    - postgresql-upgrade
```
Запуск предварительных проверок: `ansible-playbook ... --tags pre-checks`. Запуск самого обновления: `ansible-playbook ... --tags migrate,switchover`.

### 💡 Стратегии и подводные камни

- **`pg_upgrade --link` vs `--copy`**: `--link` почти мгновенен и не требует дополнительного места, но **старый кластер становится непригодным для запуска** после начала миграции. `--copy` безопаснее (позволяет откатиться простым переключением обратно), но требует свободного места, равного размеру БД.
- **Логическая репликация**: Для минимизации простоя (minutes vs seconds) иногда используют логическую репликацию между старым и новым кластером, но это значительно сложнее в автоматизации.
- **Расширения**: После `pg_upgrade` может потребоваться обновление расширений (`ALTER EXTENSION ... UPDATE`). Это стоит вынести в отдельную задачу в `switchover.yml`.
- **`ANALYZE`**: Сразу после обновления **обязательно** запустите `ANALYZE` на новом кластере для сбора свежей статистики планировщика.
- **Тестирование**: Роль для мажорного обновления обязательно нужно прогнать на стенде, максимально похожем на продакшен, с реальным дампом базы.

В итоге, структура роли для обновления смещает акцент с «приведения системы к целевому состоянию» на **«безопасное проведение рискованной операции с четкими точками контроля и возможностью отмены»**.
