# common.sh
- Colocar em todos os servidores

```bash
#!/bin/bash
set -e

echo "[COMMON] Atualizando sistema..."
apt update && apt upgrade -y

echo "[COMMON] Desabilitando swap..."
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

echo "[COMMON] Configurando módulos do kernel..."
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

echo "[COMMON] Ajustando sysctl..."
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

echo "[COMMON] Instalando dependências..."
apt install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release \
  apt-transport-https

echo "[COMMON] Instalando containerd..."
apt install -y containerd

mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml

sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml

systemctl restart containerd
systemctl enable containerd

echo "[COMMON] Adicionando repositório Kubernetes..."
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" \
  | tee /etc/apt/sources.list.d/kubernetes.list

apt update

echo "[COMMON] Instalando kubeadm, kubelet e kubectl..."
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

systemctl enable kubelet

echo "[COMMON] Finalizado com sucesso!"
```


# master.sh

- Colocar apenas no nó mestre

```bash
#!/bin/bash
set -e

POD_CIDR="192.168.0.0/16"

echo "[MASTER] Inicializando cluster Kubernetes..."
kubeadm init \
  --pod-network-cidr=${POD_CIDR} \
  --cri-socket unix:///run/containerd/containerd.sock

echo "[MASTER] Configurando kubectl para o usuário..."
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

echo "[MASTER] Instalando Calico CNI..."
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

echo "[MASTER] Aguardando nós ficarem Ready..."
sleep 10
kubectl get nodes

echo
echo "=================================================="
echo "📌 COMANDO PARA ADICIONAR WORKERS AO CLUSTER:"
kubeadm token create --print-join-command
echo "=================================================="
```



# worker.sh

- Colocar nos outros servidores que ficarem sendo worker

```bash
#!/bin/bash
set -e

echo "[WORKER] Nó preparado para ingressar no cluster Kubernetes."
echo
echo "➡️ Execute aqui o comando kubeadm join gerado no master."
echo
echo "Exemplo:"
echo "kubeadm join <MASTER_IP>:6443 --token <TOKEN> \\"
echo "  --discovery-token-ca-cert-hash sha256:<HASH>"
echo
```


#Atenção ⚠️ **Este script NÃO executa o join automaticamente**  
(boa prática: você cola o comando do master)

---

## ✅ Ordem correta de execução

- ***Em todos os nós***

```bash
chmod +x comm.sh master.sh worker.sh
sudo ./common.sh
```

- No master
```bash
sudo ./master.sh
```

- Nos workers
```bash
sudo ./worker.sh
# Depois cole o kubeadm join
```

---

## 🔍 Verificações essenciais

- Execute estes comandos apenas no master
```bash
kubectl get nodes
kubectl get pods -A
kubectl get cs
```