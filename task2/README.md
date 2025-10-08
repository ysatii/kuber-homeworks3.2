# Задание 2*. Установить HA кластер

1. Установить кластер в режиме HA.
2. Использовать нечётное количество Master-node.
3. Для cluster ip использовать keepalived или другой способ.

# Ответ 2

## подключаем еще две мастер ноды к кластеру  
Подключаем k8s-m2
# 1. Базовая подготовка
```
ssh -l lamer 158.160.198.74
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

# 2. Установка RKE2
```
curl -sfL https://get.rke2.io | sudo sh -
```
```
sudo mkdir -p /etc/rancher/rke2
```

# 3. Конфиг кластера
```
sudo tee /etc/rancher/rke2/config.yaml >/dev/null <<'EOF'
server: https://10.130.0.25:9345
token: K10d92839ebe7e26c54b06cff4c61a394d0701db0f38e6b453200b6aaa66cdcfe52::server:74d0ffe751f3231af67cd93157401fb8

node-name: k8s-m2
cni: canal
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16
tls-san:
  - 10.130.0.25      # k8s-m1
  - 10.130.0.26      # k8s-m2
  - 10.130.0.12      # k8s-m3
  - 10.128.0.100     # будущий VIP
  - k8s-m2
write-kubeconfig-mode: "0644"
EOF
```

# 4. Запуск RKE2
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
![рисунок 17](https://github.com/ysatii/kuber-homeworks3.2/blob/main/img/img_17.jpg)  
![рисунок 18](https://github.com/ysatii/kuber-homeworks3.2/blob/main/img/img_18.jpg)  
![рисунок 19](https://github.com/ysatii/kuber-homeworks3.2/blob/main/img/img_19.jpg)  

