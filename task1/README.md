## Задание 1. Установить кластер k8s с 1 master node

1. Подготовка работы кластера из 5 нод: 1 мастер и 4 рабочие ноды.
2. В качестве CRI — containerd.
3. Запуск etcd производить на мастере.
4. Способ установки выбрать самостоятельно.

## ответ 1

# Отключить swap (требование Kubernetes)

```
sudo swapoff -a
sudo sed -ri '/\sswap\s/s/^/#/' /etc/fstab
```
-----

# Загрузить необходимые модули ядра
```
cat <<'EOF' | sudo tee /etc/modules-load.d/rke2.conf
br_netfilter
overlay
EOF
sudo modprobe br_netfilter
sudo modprobe overlay
```
-----

# Настройки сетевого стека

```
cat <<'EOF' | sudo tee /etc/sysctl.d/99-rke2.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

-----

1. Установка RKE2 (Server)

```
curl -sfL https://get.rke2.io | sudo sh -
```

```
sudo systemctl enable rke2-server.service
```

ancher/rke2

```
sudo mkdir -p /etc/rancher/rke2
sudo tee /etc/rancher/rke2/config.yaml >/dev/null <<EOF
# Название ноды
node-name: k8s-m1

# Разрешённые SAN'ы для TLS
tls-san:
  - 51.250.36.28       # публичный IP
  - 10.130.0.25        # внутренний IPostname

# Явно указываем CNI
cni: canal

# Подсети кластера
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16

# Разрешаем чтение kubeconfig пользователем
write-kubeconfig-mode: "0644"
EOF
```

-----

3. Запуск кластера
```
sudo systemctl start rke2-server.service
sudo journalctl -u rke2-server -f
```

дожидаемся строки в логах
Started rke2-server.service - Rancher Kubernetes Engine v2
-----

4. Подключение kubectl и проверка состояния
### Добавляем RKE2 бинарники в PATH
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' | tee -a ~/.bashrc
source ~/.bashrc

### kubeconfig для kubectl
mkdir -p ~/.kube
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

### Проверяем состояние ноды и системных подов
kubectl get nodes -o wide
kubectl -n kube-system get pods -o wide
-----

5. Проверка CRI и etcd
### Проверяем что используется containerd
```
mkdir -p /etc/crictl.yaml
tee /etc/crictl.yaml >/dev/null <<EOF
runtime-endpoint: unix:///run/k3s/containerd/containerd.sock
image-endpoint: unix:///run/k3s/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

```
crictl info | grep -i runtimeType
```

получаем ответ 
```
        "runtimeType": "io.containerd.runc.v2",
        "runtimeType": "io.containerd.runhcs.v1",
```
containerd присутвует !
-----

### Проверяем работу etcd
```
kubectl -n kube-system get pods -l component=etcd -o wide
```
-----

6. Получение токена для присоединения worker-нод
```
sudo cat /var/lib/rancher/rke2/server/node-token
```
ответ K10d92839ebe7e26c54b06cff4c61a394d0701db0f38e6b453200b6aaa66cdcfe52::server:74d0ffe751f3231af67cd93157401fb8
-----
![рисунок 2](https://github.com/ysatii/kuber-homeworks3.2/blob/main/img/img_2.jpg)  

### 1️⃣ Проверка состояния нод

```bash
### Проверка системных компонентов
kubectl get pods -n kube-system -o wide


### Проверка control-plane
kubectl get pods -n kube-system -l tier=control-plane -o wide

### Проверка etcd
kubectl -n kube-system get pods -l component=etcd -o wide
kubectl -n kube-system logs -l component=etcd --tail=20

### Проверка CNI (Canal)
kubectl -n kube-system get pods -l k8s-app=canal -o wide

### Проверка kubelet и CRI (containerd)
systemctl status rke2-server | grep Active
sudo crictl info | grep -i runtimeType
```
![рисунок 3](https://github.com/ysatii/kuber-homeworks3.2/blob/main/img/img_3.jpg)  
![рисунок 4](https://github.com/ysatii/kuber-homeworks3.2/blob/main/img/img_4.jpg)  


```bash
7️⃣ Проверка Kubernetes API
kubectl cluster-info
kubectl version --short

8️⃣ Проверка системных сервисов
kubectl get svc -A

9️⃣ Проверка DNS внутри кластера
kubectl run dns-test --image=busybox:1.28 --restart=Never -it -- nslookup kubernetes.default

🔟 Проверка состояния компонентов
kubectl get componentstatuses