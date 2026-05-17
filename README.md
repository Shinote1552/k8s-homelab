# k8s-homelab
Домашняя лаборатория Kubernetes на VMware


# 🏠 Домашняя лаборатория Kubernetes на VMware

## 🧱 Архитектура и топология

### Виртуальные машины, они же непосредственно ноды/узлы

| Хостнейм  | Роль          | vCPU | RAM | Диск  | ОС                 |
|-----------|---------------|------|-----|-------|--------------------|
| `gantz` | master/control | 2    | 4   | 40 ГБ | Ubuntu 26.04 LTS   |
| `worker1` | worker        | 2    | 2   | 40 ГБ | Ubuntu 26.04 LTS   |

### Сетевые интерфейсы

| Интерфейс | Тип сети   | Способ назначения IP | Мастер             | Воркер             |
|-----------|------------|----------------------|--------------------|--------------------|
| `ens33`   | **NAT**        | DHCP                 | Динамический (напр. `192.168.136.132`) | Динамический (напр. `192.168.136.133`) |
| `ens37`   | **Host-Only**  | **Статический**      | `192.168.56.20/24` | `192.168.56.21/24` |

- **NAT (ens33):** Только для выхода в интернет (установка пакетов, загрузка образов).  
- **Host-Only (ens37):** Изолированная сеть для общения узлов кластера и доступа `kubectl` с хостовой Windows-машины. **Статические IP обязательны**, чтобы API-сервер и ноды всегда были доступны по статическому единственному адресу.

### Программный стек

- **Ядро Linux:** Модифицировано (см. раздел «Подготовка ОС»)
- **Container Runtime:** containerd (с включённым `SystemdCgroup`)
- **Kubernetes:** v1.29.15 (зафиксировано `apt-mark hold`)
- **Сетевой плагин (CNI):** Calico (Tigera Operator)

---

## 📦 1. Подготовка виртуальных машин

### 1.1 Создание виртуальных машин в VMware

В VMware Workstation создаём две машины (Ubuntu Server 26.04 LTS). После установки ОС **обязательно** добавляем второй сетевой адаптер в режиме **Host-Only**.

### 1.2 Настройка хостов

ВНИМАНИЕ: все внутренние сертификаты (apiserver, kubelet) будут привязаны к hostname которые будут указаны при инициализаци, поэтому если есть необходиость изменить имя лучше делать заранее!

*На мастер-узле:*
```bash
sudo hostnamectl set-hostname gantz
```
*На ворке-узле:*
```bash
sudo hostnamectl set-hostname worker1
```

### 1.3 Сетевые интерфейсы (Netplan)

На каждой ноде редактируем /etc/netplan/00-installer-config.yaml.
Ваш конфиг может слегка отличаться, это вполне нормально, я указал лишь нужные нам параметры.
Статическое ip можно задать на свое усмотрение, я выбрал x.x.56.2x, маску обязательно нужно задать.

*Мастер (192.168.56.20)*
```yaml
network:
  ethernets:
    ens33:
      dhcp4: true
    ens37:
      dhcp4: no
      addresses: [192.168.56.20/24]
```

*Воркер (192.168.56.21)*
```yaml
network:
  ethernets:
    ens33:
      dhcp4: true
    ens37:
      dhcp4: no
      addresses: [192.168.56.21/24]
```

*Применяем на всех нодах*
```bash
sudo netplan apply
```

*Проверяем на обеих нодах*
```bash
ip a show ens37
# Должен быть UP и содержать назначенный статический IP

#################################
### Проверяем с помощью ping: ###
# worker1 -> gantz
ping 192.168.56.20  # с воркера до мастера 
# gantz -> worker1
ping 192.168.56.21  # с мастера до воркера
```


## ⚙️ 2. Подготовка ОС (на всех нодах/узлах)

### 2.1 Отключение SWAP

*Kubernetes требует отключения swap для корректной работы ограничений памяти.*
```bash
sudo swapoff -a
sudo sed -i '/swap/ s/^\(.*\)$/#\1/g' /etc/fstab
sudo rm -f /swap.img   # если использовался swap-файл   
```

*Проверка:*
```bash
free -m
# Строка Swap должна быть полностью нулевая(на его строке должны быть только нули!)
# Если будут возникать проблемы при инициализации kubeadm советую перепроверить данный пункт!
```

### 2.2 Модули ядра и сеть

Контейнеризация и сетевая маршрутизация Kubernetes требуют модули overlay и br_netfilter.

*Загрузка модулей:*
```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

*Автозагрузка при старте:*
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

*Настройка параметров сети:*
```bash
cat << EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```
Пояснение:

* `bridge-nf-call-iptables` заставляет трафик между подами проходить через правила `iptables` (необходимы для Service, NetworkPolicy). Иначе трафик между контейнерами игнорировал бы правила `iptables`.

* `ip_forward` разрешает узлу маршрутизировать пакеты (поды на разных нодах могут общаться).

*Проверка:*
```bash
lsmod | grep -E "overlay|br_netfilter"
sysctl net.bridge.bridge-nf-call-iptables net.ipv4.ip_forward
# оба параметра должны быть = 1
```

## 🐳 3. Container Runtime: containerd (на всех нодах/узлах)

### 3.1 Установка и настройка

```bash
sudo apt update && sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```
Критически важно: Kubernetes kubelet использует cgroup-драйвер systemd, а containerd по умолчанию - cgroupfs. 

*Переключаем:*
```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

*Проверка:*
```bash
grep SystemdCgroup /etc/containerd/config.toml  # должно быть true
sudo systemctl status containerd
```

## ☸️ 4. Установка компонентов Kubernetes (kubeadm, kubelet, kubectl) (на всех нодах/узлах)

### 4.1 Добавление репозитория и установка (на всех узлах)
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl   # предотвращаем автоматические обновления
```

## 🧠 5. Инициализация кластера (ТОЛЬКО на мастер ноде)

### 5.1 Запуск control-plane

Флаг `--apiserver-advertise-address` обязан указывать статический IP из Host-Only сети, чтобы воркеры могли подключаться стабильно.
```bash
sudo kubeadm init --pod-network-cidr=172.16.0.0/12 --apiserver-advertise-address=192.168.56.20
```

Сохраните появившуюся команду в kubeadm join <...> - она понадобится для подключения воркер нод.

### 5.2 Конфигурация kubectl (пользователь без root)
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

*Проверка:*
```bash
kubectl get nodes
# Покажет мастер-узел в статусе NotReady (пока нет сетевого плагина)
```

### 5.3 Установка сетевого плагина Calico

Calico запускает оператор, который автоматически разворачивает pod-сеть на всех узлах.

```bash
# Установка оператора
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/tigera-operator.yaml

# Применение конфигурации (той же подсети, что и в --pod-network-cidr)
cat << EOF | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 172.16.0.0/12
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer 
metadata: 
  name: default 
spec: {}
EOF
```

*Проверка:*
```bash
kubectl get pods -n calico-system -o wide
kubectl get nodes
# Мастер должен стать Ready в течение пары минут
```

## ➕ 6. Присоединение рабочего узла (ТОЛЬКО на воркере)

### 6.1 Подготовка на воркере
Убедитесь, что выполнены все шаги 1–4 (Netplan, swap, модули, containerd, пакеты Kubernetes). Имя хоста должно быть уникальным (в примере worker1).

### 6.2 Выполнение команды join

*Используем команду, полученную при инициализации мастера.*
*На воркере:*

```bash
# То самое kubeadm join <...> из пункта 5.1
sudo kubeadm join <ip:port --token <token> --discovery-token-ca-cert-hash sha256:<hash>>
```

*Если она потерялась, генерируем новую на мастере:*
```bash
kubeadm token create --print-join-command
```

*Проверка на мастере:*
```bash
kubectl get nodes
# Должны появиться обе ноды в статусе Ready
kubectl get pods -n calico-system -o wide
# На worker1 должен быть запущен pod calico-node
```

## 🔁 7. Поведение после перезагрузки
Если все сделали правильно то при перезагрузке машин, ноды должны сами автоматически подниматься и приводится к статусу Ready.

* Swap должен оставаться отключённым, иначе kubelet упадёт. Проверьте /etc/fstab.
* Статические IP (ens37) сохраняются благодаря Netplan.
* Control-plane поднимается автоматически: containerd и kubelet в автозапуске, static pod-манифесты в /etc/kubernetes/manifests считываются kubelet.
* Рабочие нагрузки (например, nginx) восстанавливаются после запуска всех системных подов.
* Если NAT IP изменился после перезагрузки – не страшно: кластер использует Host-Only адреса.

```bash
### Быстрая проверка после перезагрузки
free -m
kubectl get nodes -o wide
kubectl get pods -n kube-system
kubectl get pods -n calico-system
```

## 🧰 8. Установка Helm
**Helm** - пакетный менеджер для Kubernetes. Устанавливается один раз на мастер-ноду и позволяет разворачивать целые стеки приложений одной командой.
На данный момент Helm не используется, но установлен для будущих экспериментов.

### 8.1 Установка из официального репозитория

```bash
sudo apt-get install curl gpg apt-transport-https --yes
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | \
  gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | \
  sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```


## 📊 9. Мониторинг (Metrics Server)

Первоначально был развёрнут полный стек Prometheus + Grafana через Helm, но он потреблял слишком много ресурсов для двух небольших виртуальных машин, вызывая перезапуски системных подов и долгую загрузку дашбордов. Поэтому он был удалён и заменён на легковесный **Metrics Server** - официальный компонент Kubernetes, предоставляющий метрики CPU и памяти через `kubectl top`.

### 9.1 Установка Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```


### 9.2 Настройка (обязательно для kubeadm-кластеров)
По умолчанию Metrics Server пытается проверить TLS‑сертификат kubelet, но в kubeadm-кластере сертификаты kubelet самоподписанные и не содержат нужных IP‑адресов в SAN. Решение - разрешить Metrics Server подключаться к kubelet без строгой проверки сертификата. Это приемлемо для учебных/домашних кластеров. 

*Добавляем флаг --kubelet-insecure-tls:*
```bash
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
```

### 9.3 Проверка
```bash
kubectl top nodes
```
*Пример вывода:*
```text
NAME      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
gantz     113m         2%     2445Mi          33%
worker1   70m          3%     435Mi           29%
```