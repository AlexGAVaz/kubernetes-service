# Предварительные действия

На всех машинах, где будет установлен кластер kubernetes, необходимо:

* Отключить swap.
* Отключить firewall.
* Отключить selinux.
* Настроить параметры ядра.
* Установить приложения.

По поводу selinux. Его можно не отключать, kubernetes и система управления контейнерами умеет его использовать.

## Отключить swap

В файле `/etc/fstab` закоментируйте строку, определяющую подключение swap пространства.

```shell
swapoff -a
```

## Отключить firewall

```shell
systemctl stop firewalld
systemctl disable firewalld
```

Убедитесь, что в фаерволе нет правил и установлены политики по умолчанию ACCEPT:

```shell
iptables -L -n
iptables -t nat -L -n
iptables -t mangle -L -n
iptables -t raw -L -n 
```

## Отключить selinux

В файле `/etc/selinux/config` установите значение переменной `SELINUX` в `disabled` или, если в дальнейшем
захотите настроить правила selinux, в `permissive`.

```shell
setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

## Настроить параметры ядра

Сначала загрузите модуль `br_netfilter`:

```shell
modprobe br_netfilter
```

Затем добавьте файл `/etc/modules-load.d/modules-kubernetes.conf`:

```shell
br_netfilter
```

В файл `/etc/sysctl.conf` добавтье следующие строки:

```conf
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_nonlocal_bind=1
```

Если планируете использовать сети ipv6, добавьте строку:

```conf
net.bridge.bridge-nf-call-ip6tables=1
```

## Установить приложения

Добавляем репозиторий kubernetes. Для этого создаём файл `/etc/yum.repos.d/kubernetes.repo`:

```shell
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
async = 1
enabled=1
baseurl = https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
gpgcheck = 1
gpgkey = https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
name = Base programs for k8s
EOF
```

_Приложения устанавливаем, но не запускаем!_

Обязательные:

* bash-completion
* python3
* tar
* containerd
* nfs-utils
* chrony
* kubectl
* kubelet
* kubeadm

```shell
dnf install -y bash-completion python3 tar containerd nfs-utils chrony kubectl kubelet kubeadm
```

Не обязательные:

* mc
* vim
* git

```shell
dnf install -y mc vim git
```

## Запуск необходимых сервисов

### NTP

Включаем NTP. Синхронизация времени на серверах кластера обязательна. Если её не включить возможны проблемы с
сертификатами.

```shell
systemctl enable chronyd
systemctl start chronyd
systemctl status chronyd
```

### syslog

Опционально включаем систему логирования rsyslog. Я предпочитаю смотреть текстовые логфайлы классического syslog, а
не копаться в бинарниках systemd. Да и потом логи системы будет легче собирать.

```shell
systemctl enable rsyslog
systemctl start rsyslog
systemctl status rsyslog
```

### containerd

Система контейнеризации containerd.

```shell
systemctl enable containerd
systemctl start containerd
systemctl status containerd
```

Добавим конфигурационный файл `/etc/crictl.yaml` для приложения управления контейнерами
[crictl](https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md):

```yaml
runtime-endpoint: "unix:///run/containerd/containerd.sock"
image-endpoint: "unix:///run/containerd/containerd.sock"
timeout: 0
debug: false
pull-image-on-create: false
disable-pull-on-run: false
```

Проверим работоспособность утилиты:

```shell
crictl images
crictl ps -a
```

Добавим аутентифекацию nexus.mediastage.tv:5002:

Для добавления настроек аутентификации nexus.mediastage.tv в файл `config.toml` для `containerd`, нужно добавить конфигурацию в раздел `[plugins."io.containerd.grpc.v1.cri".registry.configs]`. Конкретно, это включает в себя указание адреса nexus репозитория и учетных данных для аутентификации.

Конфигурационный файл должен выглядеть примерно так после добавления:

```toml
    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = ""

      [plugins."io.containerd.grpc.v1.cri".registry.auths]

      [plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."nexus.mediastage.tv:5002".auth]
          username = "You-Registry-Name"
          password = "You-Registry-Pass"

      [plugins."io.containerd.grpc.v1.cri".registry.headers]

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]

    [plugins."io.containerd.grpc.v1.cri".x509_key_pair_streaming]
      tls_cert_file = ""
      tls_key_file = ""
```

Если обнаружилось, что папка `/etc/containerd/` пуста и в ней нет базового файла `config.toml` для `containerd`, это может означать, что `containerd` использует конфигурацию по умолчанию, которая не записана в файл. Для создания и модификации конфигурационного файла `containerd`, необходимо выполнить следующие шаги:

1. **Создание базового файла `config.toml`**:
   Сгенерировать базовый файл конфигурации `containerd` с помощью самого `containerd`.

   Для этого используйте следующую команду:

   ```shell
   containerd config default > /etc/containerd/config.toml
   ```

   Эта команда создаст новый файл `config.toml` в `/etc/containerd/` с конфигурацией по умолчанию.

2. **Модификация файла `config.toml`**:
   Теперь, когда есть файл конфигурации, необходимо отредактировать его, добавив настройки аутентификации для приватного репозитория, как описано ранее.

3. **Перезапуск `containerd`**:
   После внесения изменений в конфигурацию, необходимо перезапустить службу `containerd` для применения новых настроек.

   Используйте следующую команду:

   ```shell
   sudo systemctl restart containerd
   ```

Если используется Kubernetes, важно убедиться, что любые изменения в конфигурации `containerd` совместимы с кластерами и не вызовут проблем с запуском или управлением контейнеров.

_Ещё одни интересный документ про [crictl](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/)_

## Немного автоматизации

Ansible [prepare-hosts.yaml](https://git.mediastage.tv/agavazin/kubernetes-service/-/blob/master/ansible_installation_kubeadm/services/prepare-hosts.yaml)
