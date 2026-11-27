# Обзор ключевых разделов интерфейса AWX в логике настройки пайплайна «GitLab → Ansible → AWX»: где хранится код, откуда AWX его берёт, что запускает и как это связано с GitLab.

## Projects — подключение репозитория GitLab

Центральный раздел для вашей схемы. Проект в AWX — это коллекция плейбуков, которую AWX подтягивает из системы контроля версий, то есть из Git-репозитория. Здесь указываются: [```3```](https://github.com/ansible/awx/blob/devel/docs/overview.md)

- **SCM Type** — Git, URL репозитория (например, `https://gitlab.example.com/group/project.git`), ветка или коммит. [```8```](https://baptiste.bouchereau.pro/tutorial/setup-awx-with-gitlab/)
- **Credential** типа *Source Control* — имя пользователя и токен доступа GitLab (или Deploy Key), чтобы приватный репозиторий был доступен. [```1```](https://habr.com/ru/companies/yadro/articles/866402/)
- Режим обновления: запуск по изменению (вебхук), обновление перед каждым запуском плейбука или расписание — AWX умеет синхронизировать проект из Git автоматически. [```1```](https://habr.com/ru/companies/yadro/articles/866402/)

Так плейбуки всегда остаются «живыми» в GitLab, а AWX лишь вытягивает актуальную версию.

## Credentials — учётные данные

Раздел для всех секретов, которые AWX передаёт в задачу. Для пайплайна обычно нужны минимум два типа: [```3```](https://github.com/ansible/awx/blob/devel/docs/overview.md)

- **Source Control** — токен GitLab для чтения репозитория; [```8```](https://baptiste.bouchereau.pro/tutorial/setup-awx-with-gitlab/)
- **Machine** — имя пользователя и приватный SSH-ключ (или пароль) для подключения к целевым хостам. [```1```](https://habr.com/ru/companies/yadro/articles/866402/)[```2```](https://www.pynetlabs.com/ansible-awx/)

Плюс опционально: **Vault** (пароль Ansible Vault, если часть переменных зашифрована), **Ansible Tower/Galaxy** — токен для GitLab API, если нужно вызывать AWX из пайплайна GitLab CI, и кастомные credential-типы. Секреты шифруются и не видны в плейбуках напрямую. [```8```](https://baptiste.bouchereau.pro/tutorial/setup-awx-with-gitlab/)[```2```](https://www.pynetlabs.com/ansible-awx/)

## Inventories — целевая инфраструктура

Здесь описывается, на каких хостах будет выполняться Ansible: [```3```](https://github.com/ansible/awx/blob/devel/docs/overview.md)

- статические hosts и группы — вручную через интерфейс;
- **источники (Sources)** — динамическое наполнение, например из Git-репозитория того же проекта: указываете источник «из проекта» и путь до файла inventory; [```8```](https://baptiste.bouchereau.pro/tutorial/setup-awx-with-gitlab/)
- обновление источников можно автоматизировать по расписанию или коммиту. [```1```](https://habr.com/ru/companies/yadro/articles/866402/)

В схеме с GitLab удобно держать inventory прямо в репозитории — тогда и конфигурация инфраструктуры версионируется вместе с плейбуками.

## Templates — Job Templates и Workflows

Ключевой исполняемый раздел. **Job Template** — это конфигурация одного запуска: связывает проект, конкретный плейбук (`.yml`), inventory и учётные данные. Важные поля: [```3```](https://github.com/ansible/awx/blob/devel/docs/overview.md)[```2```](https://www.pynetlabs.com/ansible-awx/)

- **Playbook** — путь до файла внутри репозитория;
- **Inventory** и **Credentials** — куда и с каким доступом;
- **Extra Variables** и **Survey** — параметризация запуска (Survey рисует пользователю форму с подсказками); [```8```](https://baptiste.bouchereau.pro/tutorial/setup-awx-with-gitlab/)
- **Enable Webhook** + Webhook Service = GitLab — генерирует Webhook URL и ключ, чтобы пайплайн или коммит в GitLab запускал шаблон автоматически; [```7```](https://www.dbi-services.com/blog/trigger-a-red-hat-ansible-automation-platform-job-workflow-from-gitlab-ci-cd-pipeline/)[```1```](https://habr.com/ru/companies/yadro/articles/866402/)
- **Execution Environment** — выбор образа, в котором выполнится плейбук (об этом ниже).

**Workflow Templates** объединяют несколько шаблонов в граф с условной логикой (успех/провал ветвлений) — удобно для многошаговых сценариев. Запуск шаблона — по кнопке «ракеты». [```4```](https://www.ansiblebyexample.com/articles/ansible-for-awx-by-examples)[```2```](https://www.pynetlabs.com/ansible-awx/)

## Execution Environments — среды исполнения

Контейнерные образы со всем нужным окружением: Ansible, коллекциями, ролями, Python-зависимостями. В разделе задаются имя образа (включая registry и тег), политика pull и credential к приватному registry; затем образ выбирается в Job Template. Это гарантирует одинаковое окружение при запуске как из AWX, так и при триггере из GitLab. [```5```](https://docs.ansible.com/projects/awx/en/24.6.1/userguide/execution_environments.html)[```1```](https://habr.com/ru/companies/yadro/articles/866402/)

## Jobs и Schedules — история и автоматизация

- **Jobs** — журнал всех запусков: вывод плейбука в реальном времени, статус по каждому хосту, ретентшн для разбора инцидентов. [```2```](https://www.pynetlabs.com/ansible-awx/)
- **Schedules** — cron-подобное расписание для шаблонов (например, ночная сверка состояния). [```4```](https://www.ansiblebyexample.com/articles/ansible-for-awx-by-examples)

## Access и Organizations — права доступа

Role-Based Access Control: команды, пользователи и гранулярные права на ресурсы. Для вашего сценария полезно дать дежурным доступ «только на запуск» шаблонов и «только на чтение» inventory — так пайплайном можно пользоваться без права менять конфигурацию. [```4```](https://www.ansiblebyexample.com/articles/ansible-for-awx-by-examples)

## Как это связывается в единый пайплайн

| Шаг | Где настраивается | Что происходит |
|---|---|---|
| 1 | GitLab | Коммит плейбуков и inventory в репозиторий |
| 2 | AWX → Projects | AWX синхронизирует код из GitLab по вебхуку или перед запуском |
| 3 | AWX → Templates | Запуск Job Template: по вебхуку, из GitLab CI через API, вручную или по расписанию |
| 4 | AWX → Jobs | Плейбук исполняется в Execution Environment на хостах из inventory, результат виден в UI |

Интеграция GitLab и AWX работает на нескольких уровнях: 
- GitLab хранит плейбуки,
- AWX синхронизирует их,
- вебхуки запускают задачи,
- AWX может возвращать статус обратно в пайплайн GitLab.[```6```](https://oneuptime.com/blog/post/2026-02-21-how-to-integrate-awx-with-gitlab/view)
