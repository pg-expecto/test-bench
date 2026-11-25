# Установка Ansible AWX на виртуальную машину Debian (K3s)

Сводная инструкция по развёртыванию AWX через `awx-operator` на однонодовом кластере K3s. Включает все шаги и исправления проблем, с которыми столкнулись при фактической установке (пользователь `test`, хост `management2`, IP `192.168.0.111`, AWX Operator 2.19.1, AWX 24.6.1).

## Требования

- Виртуальная машина с Debian 11/12: минимум 2 vCPU, **8 ГБ RAM** (при ~6 ГБ доступной памяти установка проходит, но впритык), 20+ ГБ диска
- Стабильный доступ в интернет к `quay.io` и `registry.k8s.io` — при нестабильном канале заранее настройте зеркало (раздел «Известные проблемы»)
- Пользователь с правами sudo

## Шаг 1. Установка K3s

```bash
curl -sfL https://get.k3s.io | sh -
```

Проверка:

```bash
sudo k3s kubectl get node
# NAME          STATUS   ROLES                  AGE   VERSION
# management2   Ready    control-plane,master   1m    v1.37.2+k3s1
```

## Шаг 2. Настройка kubeconfig для обычного пользователя

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown test:test ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
export KUBECONFIG=~/.kube/config        # важно: и в текущую сессию тоже!
```

> **Типичная ошибка:** добавить переменную только в `.bashrc`. Файл читается лишь при открытии нового терминала, а в текущей сессии переменная останется пустой — kubectl вернётся к `/etc/rancher/k3s/k3s.yaml` и будет падать с `open ...k3s.yaml.lock: permission denied`. Команда `export` из блока выше обязательна.

Проверка:

```bash
kubectl get nodes
kubectl config set-context --current --namespace=awx   # namespace появится на шаге 3
```

## Шаг 3. Установка AWX Operator

```bash
# Последний релиз оператора
RELEASE_TAG=$(curl -s https://api.github.com/repos/ansible/awx-operator/releases/latest | grep tag_name | cut -d '"' -f 4)
echo $RELEASE_TAG

# Namespace
export NAMESPACE=awx
kubectl create ns ${NAMESPACE}
kubectl config set-context --current --namespace=${NAMESPACE}

# Клонирование и деплой
cd ~
git clone https://github.com/ansible/awx-operator.git
cd awx-operator
git checkout $RELEASE_TAG
make deploy
```

Проверка (ожидание 2–5 минут):

```bash
kubectl get pods -n awx -w
# awx-operator-controller-manager-...   2/2   Running
```

> Warning `missing the kubectl.kubernetes.io/last-applied-configuration annotation` при деплое — безвреден, kubectl сам всё исправляет.

## Шаг 4. Исправление известной проблемы: kube-rbac-proxy

Образ `gcr.io/kubebuilder/kube-rbac-proxy` удалён из Google Container Registry, и pod оператора застревает в `ImagePullBackOff` с ошибкой `not found`. Он нужен только для метрик оператора, но без него pod не перейдёт в `Running`. Решение — скачать живой аналог и подложить под старое имя:

```bash
# 1. Скачать живой образ
sudo k3s crictl pull quay.io/brancz/kube-rbac-proxy:v0.15.0

# 2. Присвоить тег со старым именем (именно ctr, у crictl нет команды tag!)
sudo k3s ctr images tag quay.io/brancz/kube-rbac-proxy:v0.15.0 \
    gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0

# 3. Проверить (должны быть обе строки с одним sha256)
sudo k3s crictl images | grep kube-rbac-proxy

# 4. Перезапустить pod оператора
kubectl delete pod -n awx -l control-plane=controller-manager
kubectl get pods -n awx -w
```

> **Нюанс:** тег, созданный через `ctr images tag`, может слететь после перезапуска K3s или перезагрузки ВМ. Если после ребута pod снова уйдёт в `ImagePullBackOff` — просто повторите команду `k3s ctr images tag`.

## Шаг 5. Развёртывание AWX

```bash
cd ~
cat <<EOF > awx-demo.yml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-demo
spec:
  service_type: nodeport
  ingress_type: none
  hostname: awx-demo.example.com
EOF

kubectl apply -f awx-demo.yml
# awx.awx.ansible.com/awx-demo created
```

Оператор последовательно развернёт: Postgres → миграцию базы → web → task. Мониторинг:

```bash
kubectl get pods -n awx -w      # флаг -w не завершается сам — Ctrl+C для выхода
watch -n 2 free -h              # контроль памяти во втором терминале
```

Ожидаемый финал (10–40 минут, зависит от канала):

```
awx-demo-migration-24.6.1-...     Completed   ← разовая джоба, это норма
awx-demo-postgres-15-0            1/1  Running
awx-demo-web-...                  3/3  Running
awx-demo-task-...                 4/4  Running
awx-operator-controller-manager   2/2  Running
```

## Шаг 6. Вход в интерфейс

```bash
# NodePort сервиса (порт вида 30xxx в колонке PORT(S))
kubectl get svc -n awx -l "app.kubernetes.io/managed-by=awx-operator"

# Пароль администратора
kubectl get secret awx-demo-admin-password -n awx -o jsonpath="{.data.password}" | base64 --decode; echo
```

Вход: `http://192.168.0.111:<NodePort>`, логин `admin`, пароль из команды выше.

## Известные проблемы и решения

### Образ обрывается при скачивании (`connection reset by peer`)

Самый тяжёлый образ набора — `quay.io/ansible/awx-ee` (~1.9 ГБ) — канал связи рвёт посреди передачи, containerd не умеет докачивать. Симптом в `kubectl describe pod`:

```
Failed to pull image "quay.io/ansible/awx-ee:24.6.1":
read tcp 192.168.0.111:38428->2.19.183.176:443: read: connection reset by peer
```

Решение — скачать через зеркало DaoCloud и подложить под оригинальное имя:

```bash
sudo k3s crictl pull quay.m.daocloud.io/ansible/awx-ee:24.6.1
sudo k3s ctr images tag quay.m.daocloud.io/ansible/awx-ee:24.6.1 \
    quay.io/ansible/awx-ee:24.6.1
# pod подхватит образ при очередной автоматической ретрае; либо ускорить:
kubectl delete pod <имя-task-пода> -n awx
```

Постоянное решение — зеркала на уровне K3s, для всех образов сразу:

```bash
sudo tee /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "quay.io":
    endpoint:
      - "https://quay.m.daocloud.io"
  "gcr.io":
    endpoint:
      - "https://gcr.m.daocloud.io"
EOF

sudo systemctl restart k3s
```

После рестарта K3s тег `gcr.io/kubebuilder/kube-rbac-proxy` слетает — восстановить командой из Шага 4.

## Справочник по диагностике

| Симптом | Причина | Действие |
|---|---|---|
| `ImagePullBackOff` долго | образ докачивается или обрыв сети | `kubectl describe pod <имя> -n awx`, затем `crictl pull` + `ctr images tag` |
| `not found` при pull | образ удалён из реестра (gcr.io) | скачать аналог, подложить тегом |
| `connection reset by peer` | канал режет передачу больших файлов | зеркало DaoCloud (см. выше) |
| `ErrImagePull` у init-контейнера `init-receptor` | не скачался `awx-ee` | то же, что выше |
| `open k3s.yaml.lock: permission denied` | не задан `KUBECONFIG` | `export KUBECONFIG=~/.kube/config` в текущей сессии |
| `Init:0/2` висит 5–15 мин | init-контейнеры ждут базу и кеш | норма, не трогать |
| `Completed` у `migration` | разовая джоба миграции БД | норма |
| `Pending` > 5 мин без прогресса | не хватает ресурсов или тома | `describe`, проверить память |

## Удаление (если нужно начать заново)

```bash
kubectl delete awx awx-demo -n awx
kubectl delete ns awx
/usr/local/bin/k3s-killall.sh
/usr/local/bin/k3s-uninstall.sh
```

Если нужно, могу сохранить эту инструкцию файлом `.md` или адаптировать под конкретную задачу — например, под интеграцию AWX с GitLab для запуска плейбуков из CI.
