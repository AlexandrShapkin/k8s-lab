2 ноды master0 и worker0
master0:
ОС: Ubuntu 24.04.3
CPU: 2/2
RAM: 2GB
HD: 20GB - sys
Интерфейсы: ens33 (host-only) - 00:A0:A0:A0:A0:20 (172.10.10.20)

worker0:
ОС: Ubuntu 24.04.3
CPU: 2/2
RAM: 4GB
HD: 20GB - sys
Интерфейсы: ens33 (host-only) - 00:A0:A0:A0:A0:30 (172.10.10.30)

На обеих машинах:
```bash
sudo apt update
sudo apt upgrade
```

Так же для работы kubernetes нужно отключить swap:
```bash
sudo vim /etc/fstab
# там закомментировать или удалить строку
# /swap.img       none    swap    sw      0       0
sudo swapoff -a
```

Так же для удобства стоит отключить ufw, если он включен:
```bash
sudo ufw disable
sudo ufw status # можно проверить состояние
# Status: inactive - при отключенном будет так
```

Убедиться что iptables не имеет каких-то правил мешающих установлению подключений (везде должно быть `policy ACCEPT`)
```bash
sudo iptables -L -n
sudo iptables -t nat -L -n
sudo iptables -t mangle -L -n
sudo iptables -t raw -L -n 
```

Установить containerd:
```bash
sudo apt install containerd
sudo systemctl enable --now containerd
```

Включение модулей ядра:
```bash
sudo modprobe overlay -v
sudo modprobe br_netfilter -v
echo "overlay" | sudo tee -a /etc/modules
echo "br_netfilter" | sudo tee -a /etc/modules
```

- br_netfilter — этот модуль необходим для включения прозрачного маскирования и облегчения передачи трафика Virtual Extensible LAN (VxLAN) для связи между Kubernetes Pods в кластере.

- overlay — этот модуль обеспечивает необходимую поддержку на уровне ядра для правильной работы драйвера хранения overlay. По умолчанию модуль overlay может не быть включен в некоторых дистрибутивах Linux, и поэтому необходимо включить его вручную перед запуском Kubernetes.

Установить в `/etc/sysctl.conf` параметр `net.ipv4.ip_forward = 1`, затем выполнить:
```bash
sudo sysctl -p
```

Дальше как в документации:
```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

Так же для синхронизации времени:
```bash
sudo apt install chrony
sudo systemctl enable --now crony
```

Записать в `/etc/crictl.yaml`:
```bash
runtime-endpoint: "unix:///run/containerd/containerd.sock"
image-endpoint: "unix:///run/containerd/containerd.sock"
timeout: 0
debug: false
pull-image-on-create: false
disable-pull-on-run: false
```

```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml
# Изменить строку SystemdCgroup = false на true
```

Затем:
```bash
sudo systemctl restart containerd
```

## master0

Для создания кластера:
```bash
sudo kubeadm init --apiserver-advertise-address=172.10.10.20 --pod-network-cidr=192.168.0.0/16
```

- `--apiserver-advertise-address` - IP master-ноды
- `--pod-network-cidr` - подсеть для подов, в данном случае для Calico

Вывод в данном случае слудующий:
```bash
[init] Using Kubernetes version: v1.34.1
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action beforehand using 'kubeadm config images pull'
W1024 14:07:34.439432   52379 checks.go:830] detected that the sandbox image "registry.k8s.io/pause:3.8" of the container runtime is inconsistent with that used by kubeadm. It is recommended to use "registry.k8s.io/pause:3.10.1" as the CRI sandbox image.
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master0] and IPs [10.96.0.1 172.10.10.20]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost master0] and IPs [172.10.10.20 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost master0] and IPs [172.10.10.20 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "super-admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/instance-config.yaml"
[patches] Applied patch of type "application/strategic-merge-patch+json" to target "kubeletconfiguration"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests"
[kubelet-check] Waiting for a healthy kubelet at http://127.0.0.1:10248/healthz. This can take up to 4m0s
[kubelet-check] The kubelet is healthy after 501.937438ms
[control-plane-check] Waiting for healthy control plane components. This can take up to 4m0s
[control-plane-check] Checking kube-apiserver at https://172.10.10.20:6443/livez
[control-plane-check] Checking kube-controller-manager at https://127.0.0.1:10257/healthz
[control-plane-check] Checking kube-scheduler at https://127.0.0.1:10259/livez
[control-plane-check] kube-controller-manager is healthy after 1.502823531s
[control-plane-check] kube-scheduler is healthy after 2.030853307s
[control-plane-check] kube-apiserver is healthy after 3.501622175s
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node master0 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node master0 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: z5c8nu.ellb1yqzlo9e8y8m
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.10.10.20:6443 --token z5c8nu.ellb1yqzlo9e8y8m \
        --discovery-token-ca-cert-hash sha256:0bfd9cef0755795c67c2dd2feb4118b3791cd9e57fb9f6b3b7c8d1877ea2d7ee 
```

Для добавления worker-ноды интересует самая последняя часть:

```bash
...
kubeadm join 172.10.10.20:6443 --token z5c8nu.ellb1yqzlo9e8y8m \
        --discovery-token-ca-cert-hash sha256:0bfd9cef0755795c67c2dd2feb4118b3791cd9e57fb9f6b3b7c8d1877ea2d7ee 
```

Так же нужно скопировать файл конфига:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## worker0
Для добавления worker в кластер нужно просто выполнить то чтобы было получено в результате `kubeadm init` на worker0:

```bash
sudo kubeadm join 172.10.10.20:6443 --token z5c8nu.ellb1yqzlo9e8y8m \
        --discovery-token-ca-cert-hash sha256:0bfd9cef0755795c67c2dd2feb4118b3791cd9e57fb9f6b3b7c8d1877ea2d7ee 
```

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.0/manifests/tigera-operator.yaml

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.0/manifests/custom-resources.yaml
```

Далее [проверить](./troubleshooting.md#проверка-статуса-кластера) работу нод и подов. Результат должен быть схож со следующим:
```bash
kubectl get nodes
NAME      STATUS   ROLES           AGE     VERSION
master0   Ready    control-plane   3d17h   v1.34.1
worker0   Ready    <none>          3d17h   v1.34.1

kubectl get pods -A
NAMESPACE         NAME                                     READY   STATUS    RESTARTS         AGE
calico-system     calico-apiserver-6f76cf5fcb-dcp6d        1/1     Running   12 (44s ago)     91m
calico-system     calico-apiserver-6f76cf5fcb-nwn9h        1/1     Running   5 (24m ago)      91m
calico-system     calico-kube-controllers-95bf5c99-mgscc   1/1     Running   9 (21m ago)      91m
calico-system     calico-node-4r6x4                        1/1     Running   10 (14m ago)     91m
calico-system     calico-node-lpnv2                        1/1     Running   1 (16m ago)      91m
calico-system     calico-typha-78f674656c-vrx8h            1/1     Running   4 (48m ago)      91m
calico-system     csi-node-driver-km8b4                    2/2     Running   10 (8m52s ago)   91m
calico-system     csi-node-driver-pk8r9                    2/2     Running   2 (16m ago)      91m
calico-system     goldmane-7b6b78f6f-bbm9k                 1/1     Running   7 (100s ago)     91m
calico-system     whisker-5454b5464c-qdrnq                 2/2     Running   14 (2m49s ago)   91m
kube-system       coredns-66bc5c9577-hgdb8                 1/1     Running   2 (16m ago)      3d17h
kube-system       coredns-66bc5c9577-j8fqc                 1/1     Running   2 (16m ago)      3d17h
kube-system       etcd-master0                             1/1     Running   58 (16m ago)     3d17h
kube-system       kube-apiserver-master0                   1/1     Running   9 (16m ago)      3d17h
kube-system       kube-controller-manager-master0          1/1     Running   61 (16m ago)     3d17h
kube-system       kube-proxy-4tg5q                         1/1     Running   18 (12m ago)     3d15h
kube-system       kube-proxy-p5kt6                         1/1     Running   3 (16m ago)      3d17h
kube-system       kube-scheduler-master0                   1/1     Running   59 (16m ago)     3d17h
tigera-operator   tigera-operator-5b4574bd6d-gk5dw         1/1     Running   10 (12m ago)     92m
```

То есть обе ноды имеют статус Ready и все, развернутые на данный момент, поды Ready и Running.

Далее можно [проверить](./troubleshooting.md#тестирование-работы-dns-и-связи-между-подами) работу DNS. Результатом проверки послужит удачное выполнение команды `ping` по доменным именам.