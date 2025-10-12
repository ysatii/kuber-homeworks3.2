# Задание 2*. Установить HA кластер

1. Установить кластер в режиме HA.
2. Использовать нечётное количество Master-node.
3. Для cluster ip использовать keepalived или другой способ.

# Ответ 2

## подключаем еще две мастер ноды к кластеру  
### Подключаем k8s-m2 ()
### 1. Базовая подготовка
```
ssh -l ubuntu 89.169.171.232
```
```
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
```

### 2. Установка RKE2
```
curl -sfL https://get.rke2.io | sudo sh -
```
```
sudo mkdir -p /etc/rancher/rke2
```

### 3. Конфиг кластера
```
sudo tee /etc/rancher/rke2/config.yaml >/dev/null <<'EOF'
server: https://10.128.0.23:9345
token: K10d2cedb6e1d6d99409b2de45def60b0b54d09a679288183f4d91850550a60e53a::server:d31c00d46216c9474a63bc29d50814ae

node-name: k8s-m2
cni: canal
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16
tls-san:
  - 10.128.0.23      # k8s-m1
  - 10.129.0.26      # k8s-m2
  - 10.130.0.7       # k8s-m3
  - 10.128.0.100     # будущий VIP
  - k8s-m2
write-kubeconfig-mode: "0644"
EOF
```

### 4. Запуск RKE2
```
sudo systemctl enable --now rke2-server
sudo journalctl -u rke2-server -f
```
------
```
kubectl get nodes -o wide
kubectl -n kube-system get pods -l k8s-app=canal -o wide
kubectl get pods -A -o wide
```
![рисунок 17](https://github.com/ysatii/kuber-homeworks3.2/blob/main/img/img17.jpg)  
![рисунок 18](https://github.com/ysatii/kuber-homeworks3.2/blob/main/img/img18.jpg)  
![рисунок 19](https://github.com/ysatii/kuber-homeworks3.2/blob/main/img/img19.jpg)  
-----
### Подключаем k8s-m3

На k8s-m3 выполняем аналогично:

### 1. Подготовка
```
ssh -l lamer 158.160.202.198
```
```
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
```

### 2. Установка RKE2
```
curl -sfL https://get.rke2.io | sudo sh -
```
```
sudo mkdir -p /etc/rancher/rke2
```

### 3. Конфиг
```
sudo tee /etc/rancher/rke2/config.yaml >/dev/null <<'EOF'
server: https://10.128.0.23:9345
token: K10d2cedb6e1d6d99409b2de45def60b0b54d09a679288183f4d91850550a60e53a::server:d31c00d46216c9474a63bc29d50814ae

node-name: k8s-m3
cni: canal
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16
tls-san:
  - 10.128.0.23      # k8s-m1
  - 10.129.0.26      # k8s-m2
  - 10.130.0.7       # k8s-m3
  - 10.128.0.100     # будущий VIP
  - k8s-m3
write-kubeconfig-mode: "0644"
EOF
```

### 4. Запуск
```
sudo systemctl enable --now rke2-server
sudo journalctl -u rke2-server -f
```

```
kubectl get nodes -o wide
kubectl -n kube-system get pods -l k8s-app=canal -o wide
kubectl get pods -A -o wide
```

![рисунок 20](https://github.com/ysatii/kuber-homeworks3.2/blob/main/img/img20.jpg)  
![рисунок 21](https://github.com/ysatii/kuber-homeworks3.2/blob/main/img/img21.jpg)  
-----
## Кластер собран
## настроены 4 рабочих ноды и 3 управляющих





## Работаем с настройками кластреа в режиме HA и ставим keepalived

0) Проверим интерфейс и добавим VIP в SAN (на каждом master)
### имя сетевого интерфейса (обычно eth0)
```
IFACE=$(ip -o -4 route show to default | awk '{print $5}'); echo "$IFACE"
```

### убедимся, что в /etc/rancher/rke2/config.yaml есть VIP в tls-san (если нет — добавим)
```
sudo grep -q "10.128.0.100" /etc/rancher/rke2/config.yaml || \
sudo sed -i '/^tls-san:/a\  - 10.128.0.100' /etc/rancher/rke2/config.yaml
```


### перезапустим rke2-server, если файл меняли
```
sudo systemctl restart rke2-server
```
-----


## Установить haproxy + keepalived (на всех master)
1) Установить haproxy + keepalived (на всех master)
```
sudo apt-get update
sudo apt-get install -y haproxy keepalived
```

2) Конфиг HAProxy (одинаковый на всех master)
```
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
    bind 10.128.0.100:6443   # <— только VIP!
    default_backend kubernetes_masters

backend kubernetes_masters
    mode tcp
    option tcp-check
    balance roundrobin
    server k8s-m1 10.128.0.23:6443 check fall 3 rise 2
    server k8s-m2 10.129.0.26:6443 check fall 3 rise 2
    server k8s-m3 10.130.0.7:6443 check fall 3 rise 2
EOF
```
```
sudo systemctl enable --now haproxy
```


3) Конфиги keepalived (разные приоритеты)

VRRP в unicast (так надёжнее в облаках). В unicast_peer указываем IP других мастеров.

k8s-m1 (MASTER, priority 103)
```
sudo tee /etc/keepalived/keepalived.conf >/dev/null <<'EOF'
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 103
    advert_int 1

    unicast_src_ip 10.128.0.23я
    unicast_peer {
        10.129.0.26
        10.130.0.7
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
```
```
sudo sed -i "s/interface eth0/interface $(ip -o -4 route show to default | awk '{print $5}')/" /etc/keepalived/keepalived.conf
sudo systemctl enable --now keepalived
```
----


k8s-m2 (BACKUP, priority 102)
```
sudo tee /etc/keepalived/keepalived.conf >/dev/null <<'EOF'
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 102
    advert_int 1

    unicast_src_ip 10.129.0.26
    unicast_peer {
        10.128.0.23
        10.130.0.7
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
```
```
sudo sed -i "s/interface eth0/interface $(ip -o -4 route show to default | awk '{print $5}')/" /etc/keepalived/keepalived.conf
sudo systemctl enable --now keepalived
```
------



k8s-m3 (BACKUP, priority 101)
```
sudo tee /etc/keepalived/keepalived.conf >/dev/null <<'EOF'
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 101
    advert_int 1

    unicast_src_ip 10.130.0.7
    unicast_peer {
        10.129.0.26
        10.128.0.23
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
```
```
sudo sed -i "s/interface eth0/interface $(ip -o -4 route show to default | awk '{print $5}')/" /etc/keepalived/keepalived.conf
sudo systemctl enable --now keepalived
```

4) Проверки VIP и балансировки
# на любом мастере  
```
ip a | grep -A1 10.128.0.100
```
```
inet 10.128.0.100/24 scope global eth0
valid_lft forever preferred_lft forever
```
ответ от  k8s-m1 он владеет vip
-----



# API по VIP:
```
curl -k https://10.128.0.100:6443/healthz
```
# на облаках машины стоят за програмными ваервоалами и мы не сможем проверить keepalived в полной мере 


![рисунок 22](https://github.com/ysatii/kuber-homeworks3.2/blob/main/img/img22.jpg)  

# kubectl через VIP (на админ-хосте, где лежит ~/.kube/config):
```
kubectl config set-cluster default --server=https://10.128.0.100:6443
kubectl cluster-info
```

Тест failover:

# если kubectl cluster-info - выдает не верный сертификат!
исправим 


sudo sed -n '1,200p' /etc/rancher/rke2/config.yaml

# если блока tls-san нет или в нём нет VIP — запишем точный конфиг:

## Для  8s-m1 
sudo tee /etc/rancher/rke2/config.yaml >/dev/null <<'EOF'
node-name: k8s-m1  
cni: canal
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16
tls-san:
  - 127.0.0.1
  - ::1
  - 10.128.0.23      # IP k8s-m1                                              
  - 10.129.0.26      # IP k8s-m2
  - 10.130.0.7       # IP k8s-m3
  - 10.43.0.1        # ClusterIP kubernetes
  - 10.128.0.100     # **VIP**
  - k8s-m1           #  
write-kubeconfig-mode: "0644"
EOF


## Для  8s-m2
sudo tee /etc/rancher/rke2/config.yaml >/dev/null <<'EOF'
node-name: k8s-m2  
cni: canal
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16
tls-san:
  - 127.0.0.1
  - ::1
  - 10.128.0.23      # IP k8s-m1                                              
  - 10.129.0.26      # IP k8s-m2
  - 10.130.0.7       # IP k8s-m3
  - 10.43.0.1        # ClusterIP kubernetes
  - 10.128.0.100     # **VIP**
  - k8s-m2           #  
write-kubeconfig-mode: "0644"
EOF


## Для  8s-m3
sudo tee /etc/rancher/rke2/config.yaml >/dev/null <<'EOF'
node-name: k8s-m3  
cni: canal
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16
tls-san:
  - 127.0.0.1
  - ::1
  - 10.128.0.23      # IP k8s-m1                                              
  - 10.129.0.26      # IP k8s-m2
  - 10.130.0.7       # IP k8s-m3
  - 10.43.0.1        # ClusterIP kubernetes
  - 10.128.0.100     # **VIP**
  - k8s-m3           #  
write-kubeconfig-mode: "0644"
EOF



# на текущем владельце VIP
sudo systemctl stop keepalived

# через 1–3 сек VIP должен появиться на другом мастере
```
ip a | grep -A1 10.128.0.100
curl -k https://10.128.0.100:6443/healthz
```

# вернуть обратно
```
sudo systemctl start keepalived
```

![рисунок 23](https://github.com/ysatii/kuber-homeworks3.2/blob/main/img/img23.jpg)  
![рисунок 24](https://github.com/ysatii/kuber-homeworks3.2/blob/main/img/img24.jpg)  
![рисунок 25](https://github.com/ysatii/kuber-homeworks3.2/blob/main/img/img25.jpg)  







