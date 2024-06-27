# K8s operator for creating projects (namespaces, resourcequotas, rolebindings for some environments)

Данный репозиторий изначально мной склонирован из [репозитория k8s-operator-in-an-hour Павла Селиванова](https://github.com/pauljamm/k8s-operator-in-an-hour)

(дальше будут мои небольшие точечные изменения при изучении)

# Собственный Kubernetes оператор за час

Практические материалы к вебинару [Собственный Kubernetes оператор за час](https://www.youtube.com/watch?v=tFzM-2pwL8A).
> В данном репозитории лежат файлы уже созданного оператора, а в этом README
> описаны инструкции для повторения пути по созданию оператора с нуля.

## Пререквизиты

- Кластер Kubernetes (можно воспользоваться [minikube](https://minikube.sigs.k8s.io/docs/start/))
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)
- [Operator SDK](https://sdk.operatorframework.io/docs/installation/)

## Установка Operator SDK (мой пример)

На ресурсе [Operator SDK](https://sdk.operatorframework.io/docs/installation/) в момент скачивания мной была актуальна версия 1.35.0.

Установка (на linux amd64):

```bash
export ARCH=$(case $(uname -m) in x86_64) echo -n amd64 ;; aarch64) echo -n arm64 ;; *) echo -n $(uname -m) ;; esac)
export OS=$(uname | awk '{print tolower($0)}')
echo $ARCH # amd64
echo $OS # linux
export OPERATOR_SDK_DL_URL=https://github.com/operator-framework/operator-sdk/releases/download/v1.35.0
curl -LO ${OPERATOR_SDK_DL_URL}/operator-sdk_${OS}_${ARCH}
ls -alFtrh
curl -LO ${OPERATOR_SDK_DL_URL}/checksums.txt
mv -v checksums.txt operator-sdk_checksums.txt
cat operator-sdk_checksums.txt 
sha256sum operator-sdk_linux_amd64
chmod -v +x operator-sdk_${OS}_${ARCH} && sudo mv -v operator-sdk_${OS}_${ARCH} /usr/local/bin/operator-sdk
sudo chown -v root:root /usr/local/bin/operator-sdk 
sudo chmod -v 755 /usr/local/bin/operator-sdk 
ls -alFtrh /usr/local/bin/
operator-sdk version
```

## Подготовка проекта

- Создаем директорию для проекта

```bash
mkdir my-projects-operator
```

- Переходим в созданную директорию

```bash
cd my-projects-operator
```

- Инициализируем проект

```bash
ls -alF
operator-sdk init --domain kodmandvl.my.local --plugins ansible
ls -alF
```

- Создаем апи нашего оператора

```bash
operator-sdk create api \
    --group ops \
    --version v1alpha1 \
    --kind Project \
    --generate-role
```

- Вставляем в файл `config/samples/ops_v1alpha1_project.yaml` описание будущего объекта типа Project

```yaml
apiVersion: ops.kodmandvl.my.local/v1alpha1
kind: Project
metadata:
  name: myproj
spec:
  members:
    - dimka
    - myuser
  environments:
    - name: prod
      resources:
        requests:
          cpu: 4
          memory: 4Gi
        limits:
          cpu: 4
          memory: 4Gi
```

- В файле `config/crd/bases/ops.kodmandvl.my.local_projects.yaml` изменяем

```diff
     plural: projects
     singular: project
-  scope: Namespaced
+  scope: Cluster
   versions:
   - name: v1alpha1
```

- Добавлеям таски в Ansible роль. В файл `roles/project/tasks/main.yml` вставляем

```yaml
---
- name: Create a namespace
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: v1
      kind: Namespace
      metadata:
        name: "{{ ansible_operator_meta.name }}-{{ item.name }}"
        labels:
          app.kubernetes.io/managed-by: "projects-operator"
  loop: "{{ environments }}"

- name: Create a resource quota
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: v1
      kind: ResourceQuota
      metadata:
        namespace: "{{ ansible_operator_meta.name }}-{{ item.name }}"
        name: resource-quota
        labels:
          app.kubernetes.io/managed-by: "projects-operator"
      spec:
        hard:
          limits.cpu: "{{item.resources.limits.cpu}}"
          limits.memory: "{{item.resources.limits.memory}}"
          requests.cpu: "{{item.resources.requests.cpu}}"
          requests.memory: "{{item.resources.requests.memory}}"
  loop: "{{ environments }}"

- name: Create a member role building
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: rbac.authorization.k8s.io/v1
      kind: RoleBinding
      metadata:
        name: "{{ item[1] }}"
        namespace: "{{ ansible_operator_meta.name }}-{{ item[0].name }}"
        labels:
          app.kubernetes.io/managed-by: "projects-operator"
      roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: ClusterRole
        name: edit
      subjects:
        - kind: ServiceAccount
          name: "{{ item[1] }}"
          namespace: users
  with_nested:
    - "{{ environments }}"
    - "{{ members }}"
```

- Корректируем RBAC права для оператора. Файл `config/rbac/role.yaml` приводим к такому виду

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: manager-role
rules:
  ##
  ## Base operator rules
  ##
  ##
  - apiGroups:
      - ""
    resources:
      - namespaces
      - resourcequotas
    verbs:
      - create
      - delete
      - get
      - list
      - patch
      - update
      - watch
  - apiGroups:
      - rbac.authorization.k8s.io
    resources:
      - rolebindings
    verbs:
      - create
      - delete
      - get
      - list
      - patch
      - update
      - watch
  - apiGroups: 
      - rbac.authorization.k8s.io
    resources:
      - clusterroles
    verbs:
      - bind
    resourceNames:
      - edit
  ## Rules for ops.kodmandvl.my.local/v1alpha1, Kind: Project
  ##
  - apiGroups:
      - ops.kodmandvl.my.local
    resources:
      - projects
      - projects/status
      - projects/finalizers
    verbs:
      - create
      - delete
      - get
      - list
      - patch
      - update
      - watch
#+kubebuilder:scaffold:rules
```

- В файл `watches.yaml` дописываем ключ для включения слежения за cluster wide объектами

```yaml
  watchClusterScopedResources: true
```

## Сборка проекта и деплой в кластер

- Собираем образ с оператором
> Не забываем поменять `kodmandvl` на ваше реальное имя пользователя на Docker Hub или адрес вашего внутреннего реджистри

```bash
sudo su -
cd /path/to/my-projects-operator
docker login
systemctl start docker.service
make docker-build IMG=kodmandvl/my-projects-operator:v0.0.1
```

- Пушим образ в реджистри

```bash
make docker-push IMG=kodmandvl/my-projects-operator:v0.0.1
```

(дальше `make deploy` и `make undeploy` можно уже не под root выполнять)

- Деплоим все в кластер (перед этим не забыть подготовить ~/.kube/config, особенно если это всё выполняется не на ноде control-plane под root, а удалённо):

```bash
make deploy IMG=kodmandvl/my-projects-operator:v0.0.1
```

```text
$ make deploy IMG=kodmandvl/my-projects-operator:v0.0.1
cd config/manager && /path/to/my-projects-operator/bin/kustomize edit set image controller=kodmandvl/my-projects-operator:v0.0.1
/path/to/my-projects-operator/bin/kustomize build config/default | kubectl apply -f -
namespace/my-projects-operator-system created
customresourcedefinition.apiextensions.k8s.io/projects.ops.kodmandvl.my.local created
serviceaccount/my-projects-operator-controller-manager created
role.rbac.authorization.k8s.io/my-projects-operator-leader-election-role created
clusterrole.rbac.authorization.k8s.io/my-projects-operator-manager-role created
clusterrole.rbac.authorization.k8s.io/my-projects-operator-metrics-reader created
clusterrole.rbac.authorization.k8s.io/my-projects-operator-proxy-role created
rolebinding.rbac.authorization.k8s.io/my-projects-operator-leader-election-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/my-projects-operator-manager-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/my-projects-operator-proxy-rolebinding created
service/my-projects-operator-controller-manager-metrics-service created
deployment.apps/my-projects-operator-controller-manager created
```

Примечание: если уже был собран и выложен образ `my-projects-operator` и он нас устраивает, то можно сразу после скачивания этого репозитория переходить к шагу деплоя в кластер `make deploy`.

## Создание тестового Project в кластере и проверка работы оператора

- Создаем в кластере объект типа Project из подготовленного ранее файла `config/samples/ops_v1alpha1_project.yaml`

```bash
kubectl apply -f config/samples/ops_v1alpha1_project.yaml
```

- Смотрим что появился нэймспейс `myproj-prod`, ресурс квоты в нем оостветствуют тому, что мы задавали при созданиие Project, и имеются в наличии рольбиндинги для пользователей

```bash
kubectl get ns | grep -e ^NAME -e myproj-prod
```

```text
# kubectl get ns | grep -e ^NAME -e myproj-prod
NAME                             STATUS   AGE
myproj-prod                      Active   76s
```

```bash
kubectl get rolebinding -n myproj-prod
kubectl get resourcequota -n myproj-prod
```

## Добавим еще окружение test в config/samples/ops_v1alpha1_project.yaml, а ресурсы для prod уменьшим

Обновленный файл config/samples/ops_v1alpha1_project.yaml:

```yaml
apiVersion: ops.kodmandvl.my.local/v1alpha1
kind: Project
metadata:
  name: myproj
spec:
  members:
    - dimka
    - myuser
  environments:
    - name: prod
      resources:
        requests:
          cpu: 2
          memory: 2Gi
        limits:
          cpu: 2
          memory: 2Gi
    - name: test
      resources:
        requests:
          cpu: 2
          memory: 2Gi
        limits:
          cpu: 2
          memory: 2Gi
```

Примением измененный манифест и посмотрим на результаты:

```bash
kubectl apply -f config/samples/ops_v1alpha1_project.yaml
kubectl get ns | grep -e ^NAME -e myproj
kubectl get rolebinding -n myproj-prod
kubectl get resourcequota -n myproj-prod
kubectl get rolebinding -n myproj-test
kubectl get resourcequota -n myproj-test
```

## Удалим ресурс myproj и посмотрим на результаты:

```bash
kubectl get projects.ops.kodmandvl.my.local 
kubectl delete projects.ops.kodmandvl.my.local/myproj
kubectl get projects.ops.kodmandvl.my.local 
kubectl get ns | grep -e ^NAME -e myproj
```

## Заглянуть в pod my-projects-operator-controller-manager-а:

Например, запустить ansible или bash:

```bash
# Запуск ansible:
kubectl exec -it -n my-projects-operator-system deploy/my-projects-operator-controller-manager -- ansible all -l 127.0.0.1, -c local -m ping
kubectl exec -it -n my-projects-operator-system deploy/my-projects-operator-controller-manager -- ansible --version
## Запуск bash:
kubectl exec -it -n my-projects-operator-system deploy/my-projects-operator-controller-manager -- bash
```

## Очистка окружения

Для удаления оператора, всех созданных им объектов и crd из кластера выполните команду:

```bash
kubectl api-resources | grep projects.*ops.kodmandvl.my.local
make undeploy IMG=kodmandvl/k8s-operator-in-an-hour:v0.0.1
kubectl api-resources | grep projects.*ops.kodmandvl.my.local
```

```text
$ kubectl api-resources | grep projects.*ops.kodmandvl.my.local
projects                                         ops.kodmandvl.my.local/v1alpha1     false        Project
$ make undeploy IMG=kodmandvl/k8s-operator-in-an-hour:v0.0.1
/path/to/my-projects-operator/bin/kustomize build config/default | kubectl delete -f -
namespace "my-projects-operator-system" deleted
customresourcedefinition.apiextensions.k8s.io "projects.ops.kodmandvl.my.local" deleted
serviceaccount "my-projects-operator-controller-manager" deleted
role.rbac.authorization.k8s.io "my-projects-operator-leader-election-role" deleted
clusterrole.rbac.authorization.k8s.io "my-projects-operator-manager-role" deleted
clusterrole.rbac.authorization.k8s.io "my-projects-operator-metrics-reader" deleted
clusterrole.rbac.authorization.k8s.io "my-projects-operator-proxy-role" deleted
rolebinding.rbac.authorization.k8s.io "my-projects-operator-leader-election-rolebinding" deleted
clusterrolebinding.rbac.authorization.k8s.io "my-projects-operator-manager-rolebinding" deleted
clusterrolebinding.rbac.authorization.k8s.io "my-projects-operator-proxy-rolebinding" deleted
service "my-projects-operator-controller-manager-metrics-service" deleted
deployment.apps "my-projects-operator-controller-manager" deleted
$ kubectl api-resources | grep projects.*ops.kodmandvl.my.local
```
