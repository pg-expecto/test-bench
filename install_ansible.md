# Установка Ansible на виртуальную машину Debian

Сводная инструкция по развёртыванию Ansible на Debian. Контекст тот же: пользователь `test`, хост `management2`, IP `192.168.0.111`, Debian 12. Ansible ставится на управляющую машину (с неё вы будете управлять остальными) и не требует агентов на целевых хостах — достаточно SSH.

## Требования

- Debian 12 (Bookworm), Python 3.9+ (в Debian 12 уже есть)
- Права sudo
- Минимум 1 ГБ RAM и пара гигабайт диска — Ansible нетребователен, это не GitLab и не AWX

## Выбор способа установки

| Способ | Версия | Кому подходит |
|---|---|---|
| `pipx` (виртуальное окружение) | всегда актуальная | рекомендую: свежие модули и коллекции |
| `apt install ansible` | 2.14 из репозитория Debian | для простых задач, если версий не касаетесь |

Репозиторий Debian обновляет пакеты редко, а версии Ansible влияют на поддержку новых модулей и синтаксис плейбуков. Для работы в связке с AWX и GitLab лучше свежая версия — поэтому основной путь ниже через `pipx`, а `apt` — запасной.

## Шаг 1. Основной способ: установка через pipx

```bash
# Зависимости Python
sudo apt update
sudo apt install -y python3-venv python3-pipx sshpass

# Сам Ansible (изолированно в своём окружении)
pipx install --include-deps ansible

# Делаем доступным в PATH
pipx ensurepath
source ~/.bashrc
```

> **Зачем `--include-deps`:** пакет `ansible` — это «зонтик» над `ansible-core`; без этого флага в PATH попадёт только ядро без набора модулей.

Проверка:

```bash
ansible --version
# ansible [core 2.x.x] ...
#   python version = 3.11.x
```

## Шаг 2. Запасной способ: через apt

Если pipx недоступен или канал до PyPI капризничает (учитывая вашу историю с обрывами при скачивании awx-ee — вполне возможно):

```bash
sudo apt install -y ansible
ansible --version
```

Минус — версия 2.14. Для обучения и базовых плейбуков её достаточно.

## Шаг 3. Структура проекта и первый запуск

Создайте рабочий проект:

```bash
mkdir -p ~/ansible-lab && cd ~/ansible-lab

# Инвентарь — список управляемых хостов
cat <<EOF > inventory.ini
[linux]
management2 ansible_host=192.168.0.111 ansible_user=test
# Добавьте другие ВМ по образцу:
# web1 ansible_host=192.168.0.112 ansible_user=test
EOF

# Конфигурация
cat <<EOF > ansible.cfg
[defaults]
inventory = ./inventory.ini
host_key_checking = False
EOF
```

Проверка связи (ping по SSH, не ICMP):

```bash
ansible linux -m ping
```

Ожидаемый ответ:

```
management2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Если вернулась ошибка `Permission denied` — на целевом хосте не настроен вход по SSH-ключу:

```bash
ssh-keygen -t ed25519          # если ключа ещё нет (Enter на все вопросы)
ssh-copy-id test@192.168.0.111
```

## Шаг 4. Тестовый плейбук

```bash
cat <<EOF > test.yml
---
- name: Проверка Ansible
  hosts: linux
  become: true
  tasks:
    - name: Установить htop
      ansible.builtin.apt:
        name: htop
        state: present
        update_cache: true
EOF

ansible-playbook test.yml
```

Итог должен выглядеть как `ok=X changed=1 ... failed=0`. Повторный запуск даст `changed=0` — это идемпотентность, главное свойство Ansible: повторное выполнение не ломает состояние.

## Известные проблемы и решения

### Обрывы при установке коллекций (`ansible-galaxy collection install`)

Галактика Ansible (`galaxy.ansible.com`) качает большие архивы — у вас канал уже демонстрировал `connection reset by peer` на файлах в гигабайт. Коллекции легче, но если обрывы начнутся, просто повторите команду — `ansible-galaxy` докачивает. При системных проблемах можно перейти на зеркало через файл `ansible.cfg`:

```ini
[galaxy]
server = https://galaxy.ansible.com   # подмените на доступное зеркало, если появится
```

### Ошибка `sudo: a password is required`

При `become: true` Ansible просит sudo-пароль интерактивно. Если это неудобно в автоматизации, разрешите пользователю sudo без пароля (осознанно, на своей лабораторной ВМ):

```bash
sudo visudo
# добавить строку:
test ALL=(ALL) NOPASSWD: ALL
```

### Криптографическая ошибка при первом подключении

`host_key_checking = False` в `ansible.cfg` снимает запрос подтверждения отпечатка хоста. Для лаборатории нормально; в продакшене — уберите и управляйте known_hosts.

### Конфликт версий pipx и системного python

Не ставьте Ansible через `pip install --user ansible` глобально — это засоряет системный Python. pipx изоляцию решает, именно поэтому он в инструкции.

## Справочник по диагностике

| Симптом | Причина | Действие |
|---|---|---|
| `Permission denied` при `-m ping` | нет SSH-ключа на целевом хосте | `ssh-copy-id user@host` |
| `sudo: a password is required` | become без пароля | ввести пароль флагом `--ask-become-pass` или настроить NOPASSWD |
| обрывы `ansible-galaxy install` | нестабильный канал | повторить команду — докачивает |
| `WARNING: ansible-playbook not found` после pipx | PATH не обновлён | `pipx ensurepath` + `source ~/.bashrc` |
| `changed=0` на повторном запуске | это не ошибка | идемпотентность, так и должно быть |

## Удаление

```bash
pipx uninstall ansible        # основной способ
# или
sudo apt purge ansible        # запасной способ
```

## Логика связки на вашей инфраструктуре

Теперь на `management2` складывается полная картина: **K3s + AWX** — оркестрация и веб-интерфейс для запуска плейбуков, **GitLab** — хранение кода и CI/CD, **Ansible** — инструмент исполнения. Классический сценарий: плейбуки лежат в репозитории GitLab, пайплайн или AWX запускает их на целевые хосты.

Следующий шаг, который я бы рекомендовала — установка `gitlab-runner` на эту же ВМ и первый пайплайн, запускающий плейбук из репозитория. Скажите, когда будете готовы — подготовлю инструкцию под эту часть 🙂
