# Ansible Playbooks для настройки Kubernetes HA (multi-master) в CentOS 7

Этот репозиторий содержит Ansible Playbooks для настройки Kubernetes HA в CentOS 7. Для создания playbook-ов использовалась в основномдокументация Kubeadm и рекомендации из других источников. Некоторые playbooks могут использоваться отдельно или как один playbook для развертывания полноценного кластера высокой доступности.

---

## Table of Contents

1. [Схема развёртывания](#deployments_scheme)
2. [Предварительные требования](#requirements)
3. [Подготовка окружения](#prepare)
4. [Установка кластера Kubernetes HA multi-master](#k8s_ha_installation)
5. [Добавление рабочей ноды в кластер](#add_worker_node)
6. [Замечания по Kubernetes Dashboard](#dashboard_comment)
7. [Пароль администратора Grafana](#grafana_password)

---

## <a name="deployments_scheme">Схема развёртывания</a>

![Kubernates HA cluster (multi-master) ](images/kubernetes_ha_cluster.png)

## <a name="requirements">Предварительные требования</a>

### Masters

- Физическая или виртуальная система
- OC: CentOS 7.7 (2003) или более поздней версии с опцией установки "Minimal"
- Минимум 2 vCPU (дополнительные настоятельно рекомендуются).
- Минимум 4 ГБ ОЗУ (настоятельно рекомендуется дополнительная память, особенно если на мастерах располагается etcd).
- Минимум 30 ГБ на жестком диске для файловой системы, содержащей /var/.
- Минимум 1 ГБ на жестком диске для файловой системы, содержащей /usr/local/bin/.
- Минимум 1 ГБ на жестком диске для файловой системы, содержащей временный каталог системы.

### Workers

- Физическая или виртуальная система
- OC: CentOS 7.7 (2003) или более поздней версии с опцией установки "Minimal"
- Минимум 4 vCPU (дополнительные настоятельно рекомендуются).
- Минимум 8 ГБ ОЗУ (настоятельно рекомендуется дополнительная память, особенно если на мастерах располагается etcd).
- Минимум 30 ГБ на жестком диске для файловой системы, содержащей /var/.
- Минимум 1 ГБ на жестком диске для файловой системы, содержащей /usr/local/bin/.
- Минимум 1 ГБ на жестком диске для файловой системы, содержащей временный каталог системы.

### Ansible host

- Физическая или виртуальная система
- OC: CentOS 7.7 (2003) или более поздней версии
- Установленный Ansibe 2.9.10

## <a name="prepare">Подготовка окружения</a>

**Клонировать репозиторий**: На машине, которая будет использоваться в качестве менеджера (это может быть ноутбук пользователя или любая другая машина, которая имеет доступ по ssh к целевым машинам):

```shell
git clone git@gitlab.com:nwtn/infrastructure/k8s-ha-cluster-ansible.git
cd k8s-ha-cluster-ansible
```

**Создать `inventory/<название среды>`**. В созданном файлt перечислить свои машины и конфигурационные параметры кластера Kubernetes:

```shell
cp inventory/k8s_cluster.example <mycluster>.ini
```

Также необходимо обновить раздел `vars`:

- настройка версии Kubernetes
- настройка сетевого модуля pod (настройка по умолчанию для flannel)
- настройка Docker на использование частного репозитория образов

Для проверки доступности всех машин, можно выполнить команду:

```shell
ansible -m ping all -i inventory/<mycluster>.ini
```

Предварительно необходимо сгенерировать ssh-ключ на машине с ansible и разместить его на целевых машинах:

```shell
ssh-keygen -t rsa
for i in {1..6}; do ssh-copy-id root@xxx.xxx.xxx.xx$i; done
```

## <a name="k8s_ha_installation">Установка кластера Kubernetes HA (multi-master)</a>

После всех настроек необходимо запустить k8s playbook, чтобы создать кластер. Этот playbook включает в себя следующие playbook-и, которые могут быть выполнены отдельно:

| Playbook               | Краткое описание                                                                                                                                                                                        |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| prepare_hosts.yml      | Предварительная настройка ОС:<br><li>настройка hostname</li><li>установка пакетов</li><li>отключение лишних сервисов</li><li>отключение swap</li><li>отключение selinux</li>                            |
| installing_runtime.yml | Установка среды выполнения контейнеров (Docker)                                                                                                                                                         |
| deploy_ha_layer.yml    | <li>Установка haproxy</li><li>Установка keepalived и настройка управления vip для Kubernetes API Server</li>                                                                                            |
| deploy_kubernetes.yml  | <li>Установка kubeadm, kubelet and kubectl</li><li>Создание кластера Kubermetes HA (multi-master)</li>                                                                                                  |
| helm_installation.yml  | Установка менеджер пактов Helm 3 для Kubernetes                                                                                                                                                         |
| helm_apps.yml          | Разворачивание в кластере Kubernetes с помощью Helm 3 системных приложений: <br><li>ingress-nginx</li><li>kubernetes-dashboard</li><li>metrics-server</li><li>prometheus operator</li><li>elkstack</li> |
| add_worker_node.yml    | Добавление в кластер Kubernetes рабочих нод                                                                                                                                                             |

Запуск playbook осуществляется командой:

```shell
ansible-playbook -i inventory/<mycluster>.ini playbooks/k8s.yml
```

## <a name="add_worker_node">Добавление рабочей ноды в кластер</a>

Для добавления новой Worker node необходимо:

- добавить описание ноды в `inventory/<mycluster>.ini` в ссекцию `[k8s_workers]`

    ```bash
    my-new-worker-hostname3.mydomain.com ansible_host=<ip1>
    ```

- протестировать сделанные изменения (dry-run mode) для добавленной ноды

    ```shell
    ansible-playbook -i inventory/<mycluster>.ini --limit k8s-node3.test.lab playbooks/add_worker_node.yml --check
    ```

- если нет ошибок, выполнить добавлени ноды

    ```shell
    ansible-playbook -i inventory/<mycluster>.ini --limit k8s-node3.test.lab playbooks/add_worker_node.yml
    ```

## <a name="dashboard_comment">Замечания по Kubernetes Dashboard</a>

Получение токена для dashboard

```shell
kubectl describe secret -n kube-dashboard $(kubectl get secret -n kube-dashboard | grep  kubernetes-dashboard  | awk '{print $1}') | grep token | awk '{print $2}'
```

По умолчанию, при установке включаются права "только на чтение", поэтому для включения полных прав (**не рекомендуется для промышленной среды**) необходимо выполнить команду на одном из kubernetes master:

```bash
cat  <<EOF | kubectl apply -f -
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kube-dashboard

EOF
```

## <a name="grafana_password"> Пароль администратора Grafana</a>

Для получения текущего пароля администратора Grafana выполнить команду:

```shell
kubectl --namespace kube-monitoring get secret prometheus-operator-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```
