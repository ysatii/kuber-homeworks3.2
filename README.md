# Домашнее задание к занятию «Установка Kubernetes» - Мельник Юрий Александрович

### Цель задания

Установить кластер K8s.

### Чеклист готовности к домашнему заданию

Развёрнутые ВМ с ОС Ubuntu 20.04-lts.


### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Инструкция по установке kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/).
2. [Документация kubespray](https://kubespray.io/).

-----

## подготовка
создадим необходимы машины в яндекс облаке 

![рисунок 1](https://github.com/ysatii/kuber-homeworks3.2/blob/main/img/img_1.jpg)  

## Задание 1. Установить кластер k8s с 1 master node

1. Подготовка работы кластера из 5 нод: 1 мастер и 4 рабочие ноды.
2. В качестве CRI — containerd.
3. Запуск etcd производить на мастере.
4. Способ установки выбрать самостоятельно.









 

------




----

----



k8s-w4 (10.130.0.9)

```
sudo hostnamectl set-hostname k8s-w4
sudo swapoff -a && sudo sed -ri '/\sswap\s/s/^/#/' /etc/fstab
echo -e "br_netfilter\noverlay" | sudo tee /etc/modules-load.d/rke2.conf
sudo modprobe br_netfilter && sudo modprobe overlay
cat <<'EOF' | sudo tee /etc/sysctl.d/99-rke2.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
sudo sysctl --system

curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sudo sh -
sudo mkdir -p /etc/rancher/rke2
sudo tee /etc/rancher/rke2/config.yaml >/dev/null <<EOF
server: https://10.128.0.18:9345
token: K10db64f6080d260611d38068f0dc09fcc5eeb7fe0505d8952514f19a83e3706856::server:f26e27b1cff1339a07224daa6b83b33e
node-name: k8s-w4
EOF
sudo systemctl enable --now rke2-agent.service
```

----

Проверки (выполняй на мастере k8s-m1)
kubectl get nodes -o wide
kubectl -n kube-system get pods -l k8s-app=canal -o wide
kubectl get pods -A -o wide

 













## Дополнительные задания (со звёздочкой)

**Настоятельно рекомендуем выполнять все задания под звёздочкой.** Их выполнение поможет глубже разобраться в материале.   
Задания под звёздочкой необязательные к выполнению и не повлияют на получение зачёта по этому домашнему заданию. 
------


------
### Задание 2*. Установить HA кластер

1. Установить кластер в режиме HA.
2. Использовать нечётное количество Master-node.
3. Для cluster ip использовать keepalived или другой способ.

Подключаем k8s-m2


# 1. Базовая подготовка
sudo hostnamectl set-hostname k8s-m2
sudo swapoff -a && sudo sed -ri '/\sswap\s/s/^/#/' /etc/fstab
echo -e "br_netfilter\noverlay" | sudo tee /etc/modules-load.d/rke2.conf
sudo modprobe br_netfilter && sudo modprobe overlay
cat <<'EOF' | sudo tee /etc/sysctl.d/99-rke2.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
sudo sysctl --system

# 2. Установка RKE2
curl -sfL https://get.rke2.io | sudo sh -
sudo mkdir -p /etc/rancher/rke2

# 3. Конфиг кластера
sudo tee /etc/rancher/rke2/config.yaml >/dev/null <<'EOF'
server: https://10.128.0.18:9345
token: K10db64f6080d260611d38068f0dc09fcc5eeb7fe0505d8952514f19a83e3706856::server:f26e27b1cff1339a07224daa6b83b33e

node-name: k8s-m2
cni: canal
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16
tls-san:
  - 10.128.0.18      # k8s-m1
  - 10.130.0.30      # k8s-m2
  - 10.130.0.36      # k8s-m3
  - 10.128.0.100     # будущий VIP
  - k8s-m2
write-kubeconfig-mode: "0644"
EOF

# 4. Запуск RKE2
sudo systemctl enable --now rke2-server
sudo journalctl -u rke2-server -f
------



Подключаем k8s-m3

На k8s-m3 выполняем аналогично:

# 1. Подготовка
sudo hostnamectl set-hostname k8s-m3
sudo swapoff -a && sudo sed -ri '/\sswap\s/s/^/#/' /etc/fstab
echo -e "br_netfilter\noverlay" | sudo tee /etc/modules-load.d/rke2.conf
sudo modprobe br_netfilter && sudo modprobe overlay
cat <<'EOF' | sudo tee /etc/sysctl.d/99-rke2.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
sudo sysctl --system

# 2. Установка RKE2
curl -sfL https://get.rke2.io | sudo sh -
sudo mkdir -p /etc/rancher/rke2

# 3. Конфиг
sudo tee /etc/rancher/rke2/config.yaml >/dev/null <<'EOF'
server: https://10.128.0.18:9345
token: K10db64f6080d260611d38068f0dc09fcc5eeb7fe0505d8952514f19a83e3706856::server:f26e27b1cff1339a07224daa6b83b33e

node-name: k8s-m3
cni: canal
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16
tls-san:
  - 10.128.0.18
  - 10.130.0.30
  - 10.130.0.36
  - 10.128.0.100
  - k8s-m3
write-kubeconfig-mode: "0644"
EOF

# 4. Запуск
sudo systemctl enable --now rke2-server
sudo journalctl -u rke2-server -f


--------------------------------


0) Проверим интерфейс и добавим VIP в SAN (на каждом master)
# имя сетевого интерфейса (обычно eth0)
IFACE=$(ip -o -4 route show to default | awk '{print $5}'); echo "$IFACE"

# убедимся, что в /etc/rancher/rke2/config.yaml есть VIP в tls-san (если нет — добавим)
sudo grep -q "10.128.0.100" /etc/rancher/rke2/config.yaml || \
sudo sed -i '/^tls-san:/a\  - 10.128.0.100' /etc/rancher/rke2/config.yaml

# перезапустим rke2-server, если файл меняли
sudo systemctl restart rke2-server

1) Установить haproxy + keepalived (на всех master)
sudo apt-get update
sudo apt-get install -y haproxy keepalived

2) Конфиг HAProxy (одинаковый на всех master)
sudo tee /etc/haproxy/haproxy.cfg >/dev/null <<'EOF'
global
    log /dev/log local0
    maxconn 4096
defaults
    log     global
    mode    tcp
    option  tcplog
    timeout connect 5s
    timeout client  1m
    timeout server  1m

frontend kubernetes_api
    bind 0.0.0.0:6443
    default_backend kubernetes_masters

backend kubernetes_masters
    mode tcp
    option tcp-check
    balance roundrobin
    server k8s-m1 10.128.0.18:6443 check fall 3 rise 2
    server k8s-m2 10.130.0.30:6443 check fall 3 rise 2
    server k8s-m3 10.129.0.6:6443  check fall 3 rise 2
EOF

sudo systemctl enable --now haproxy

3) Конфиги keepalived (разные приоритеты)

VRRP в unicast (так надёжнее в облаках). В unicast_peer указываем IP других мастеров.

k8s-m1 (MASTER, priority 103)
sudo tee /etc/keepalived/keepalived.conf >/dev/null <<'EOF'
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 103
    advert_int 1

    unicast_src_ip 10.128.0.18
    unicast_peer {
        10.130.0.30
        10.129.0.6
    }

    authentication {
        auth_type PASS
        auth_pass 42secret
    }

    virtual_ipaddress {
        10.128.0.100/24
    }
}
EOF
sudo sed -i "s/interface eth0/interface $(ip -o -4 route show to default | awk '{print $5}')/" /etc/keepalived/keepalived.conf
sudo systemctl enable --now keepalived

k8s-m2 (BACKUP, priority 102)
sudo tee /etc/keepalived/keepalived.conf >/dev/null <<'EOF'
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 102
    advert_int 1

    unicast_src_ip 10.130.0.30
    unicast_peer {
        10.128.0.18
        10.129.0.6
    }

    authentication {
        auth_type PASS
        auth_pass 42secret
    }

    virtual_ipaddress {
        10.128.0.100/24
    }
}
EOF
sudo sed -i "s/interface eth0/interface $(ip -o -4 route show to default | awk '{print $5}')/" /etc/keepalived/keepalived.conf
sudo systemctl enable --now keepalived

k8s-m3 (BACKUP, priority 101)
sudo tee /etc/keepalived/keepalived.conf >/dev/null <<'EOF'
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 101
    advert_int 1

    unicast_src_ip 10.129.0.6
    unicast_peer {
        10.128.0.18
        10.130.0.30
    }

    authentication {
        auth_type PASS
        auth_pass 42secret
    }

    virtual_ipaddress {
        10.128.0.100/24
    }
}
EOF
sudo sed -i "s/interface eth0/interface $(ip -o -4 route show to default | awk '{print $5}')/" /etc/keepalived/keepalived.conf
sudo systemctl enable --now keepalived

4) Проверки VIP и балансировки
# на любом мастере
ip a | grep -A1 10.128.0.100

# API по VIP:
curl -k https://10.128.0.100:6443/healthz

# kubectl через VIP (на админ-хосте, где лежит ~/.kube/config):
kubectl config set-cluster default --server=https://10.128.0.100:6443
kubectl cluster-info


Тест failover:

# на текущем владельце VIP
sudo systemctl stop keepalived
# через 1–3 сек VIP должен появиться на другом мастере
ip a | grep -A1 10.128.0.100
curl -k https://10.128.0.100:6443/healthz
# вернуть обратно
sudo systemctl start keepalived











### Правила приёма работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl get nodes`, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.
