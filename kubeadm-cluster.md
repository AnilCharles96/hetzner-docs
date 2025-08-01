# Kubeadm



## Create first control plane

### Pre-requisites


### net.ipv4.ip_forward=1

```
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

```

```
sudo sed -i '/ swap / s/^/#/' /etc/fstab

swapoff -a

# Download containerd
wget https://github.com/containerd/containerd/releases/download/v2.1.3/containerd-2.1.3-linux-amd64.tar.gz

tar Cxzvf /usr/local containerd-2.1.3-linux-amd64.tar.gz

# runc
wget https://github.com/opencontainers/runc/releases/download/v1.3.0/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc

```

### Containerd systemd service

```
mkdir -p /usr/local/lib/systemd/system/

nano /usr/local/lib/systemd/system/containerd.service

```

### Copy below to above

```
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target dbus.service

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```

### Start containerd
```
systemctl daemon-reload
systemctl enable --now containerd
```


### Install CNI plugins
```
mkdir -p /opt/cni/bin

wget https://github.com/containernetworking/plugins/releases/download/v1.7.1/cni-plugins-linux-amd64-v1.7.1.tgz

tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.7.1.tgz

```

### Install kubeadm

```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet


```

### Failed to create pod sandbox: open /run/systemd/resolve/resolv.conf: no such file or directory
```
mkdir -p /run/systemd/resolve
touch /run/systemd/resolve/resolv.conf
```

###  error: open /var/lib/kubelet/config.yaml: no such file or directory
```
kubeadm init phase kubelet-start
```

### hostname "controlplane-1" could not be reached

```
sudo kubeadm reset -f
```

### Kubeadm reset -f freezes
```
sudo rm -rf /etc/kubernetes/
sudo rm -rf /var/lib/etcd/
sudo rm -rf /var/lib/kubelet/
sudo rm -rf /var/lib/cni/
sudo rm -rf /etc/cni/
sudo rm -rf /opt/cni/
sudo rm -rf /var/run/kubernetes/
sudo rm -rf ~/.kube/
````

###  \"runc\": executable file not found in $PATH"

```
wget https://github.com/opencontainers/runc/releases/download/v1.3.0/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc
```


### Install control plane

```
kubeadm init --skip-phases=addon/kube-proxy --upload-certs \
--control-plane-endpoint=10.0.0.102:6443 --node-name controlplane-1


```

## Get the cluster config

```

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

# Install Helm
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

# Install Cilium

## Get API SERVER

```
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'

API_SERVER_IP=10.0.0.102


helm repo add cilium https://helm.cilium.io/

helm upgrade --install cilium cilium/cilium --version 1.17.5 \
    --namespace kube-system -f cilium.yaml

kubectl -n kube-system rollout restart daemonset/cilium
kubectl -n kube-system rollout restart deployment/cilium-operator
kubectl -n kube-system rollout restart deployment/hubble-relay



```


## Cilium CLI





```
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

```
cilium status --wait
```

# Joining control plane/worker node

## Token

```
kubeadm token create
```

## Discovery Token

```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
                                  openssl rsa -pubin -outform der 2>/dev/null | \
                                  sha256sum | awk '{print $1}'

```

## Certificate key

```
 kubeadm init phase upload-certs --upload-certs
```

## Troubleshoot
```
journalctl -u kubelet -n 5
```

## Join second control plane

```
kubeadm join 172.16.153.145:6443 --token qgtle3.kx217h6pqaq1kxkn \
--discovery-token-ca-cert-hash sha256:5d33c626ad8e29a448b205713189edd9f7fcae4d12f94d2b07bd182d375997e8 \
--certificate-key daada77571cb4de6d7dcb76c82fa06829ac1852e3bd054901fc7b545f6c21251 \
--node-name controlplane-2 \
--control-plane
```

## Join worker node

```

 kubeadm join 172.16.153.147:6443 --token 976tt8.tbd4xuoj59ut33p6 \
	--discovery-token-ca-cert-hash sha256:5e645e2e5fa3542a68b78302ae8d41e8fbda53a4f6d3956c9602e264f6a444ed \
	--certificate-key d09f23d76e307981af26b72ec6d7d8fe3546c759a37a2561dfad0e0edd4a092c



```

# k9s
```
wget https://github.com/derailed/k9s/releases/download/v0.50.6/k9s_linux_amd64.deb

dpkg -i k9s_linux_amd64.deb
```

# Istio
```
curl -L https://istio.io/downloadIstio | sh -

cd istio-1.26.2

export PATH=$PWD/bin:$PATH

kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
{ kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.3.0" | kubectl apply -f -; }

```

# Single dedicated server 2 control plane

```
kubeadm init --skip-phases=addon/kube-proxy --node-name controlplane-1


kubeadm join 10.0.0.102:6443 --token wmi7iy.d273hzbajtk6mhzm \
							--node-name controlplane-2 \
                       --control-plane \
                       --discovery-token-ca-cert-hash sha256:9c9790c3311e7901145d061fd5c24b8e0284b449c5299dda278322cc2fd8fe69


kubeadm join 10.0.0.102:6443 --token x75ew3.vbcnmebtomupifc3 --control-plane --node-name controlplane-2 --discovery-token-ca-cert-hash sha256:270476e050fed5d59d0758587e0017ad7678a0c7722f3449aa9e8517ab78b812 --certificate-key d12372c83bed25293d0745e973bacf0c1fee198798084d2067b8a689569345bf

# worker
kubeadm join 10.0.0.102:6443 --token x75ew3.vbcnmebtomupifc3 \
	--discovery-token-ca-cert-hash sha256:270476e050fed5d59d0758587e0017ad7678a0c7722f3449aa9e8517ab78b812 --node-name worker-1

```


# Local path provisioner for pvc
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
provisioner: rancher.io/local-path
volumeBindingMode: Immediate



kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml


kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```