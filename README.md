# Kubernetes + containerd Setup (Fedora, Debian/Ubuntu, Arch/Manjaro)

Este README contém instruções detalhadas sobre como configurar um cluster Kubernetes de nó único com containerd como runtime em diferentes distribuições Linux: Fedora, Debian/Ubuntu e Arch/Manjaro. Também cobre comandos essenciais do Kubernetes.

## Pré-Requisitos
- Conexão com a internet
- Acesso root/sudo
- curl, wget, dnf/dnf5 (Fedora), apt (Debian/Ubuntu), pacman (Arch/Manjaro)

## Configuração Inicial
- Desabilite o swap, o Kubernetes não funciona corretamente com o swap ligado. O kubelet, componente que gerencia os pods no nó, não lida bem com o swap habilitado, inteferindo no gerenciamento de memória dos contêineres, causando comportamentos imprevisíveis e comprometendo a estabilidade do cluster.
### Desabilitar o swap e zram para Kubernetes
1. Desativar swap padrão (fstab)
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
2. Verificar swap ativo
```bash
free -h
swapon --summary
```
3. Desativar ZRAM permanente (caso ativo)
- Verificar se o zram está ativo
```bash
swapon --summary
```
- Opção 1: Desabilitar serviço systemd
```bash
sudo systemd disable --now systemd-zram-setup@zram0.service
```
- Opção 2: Criar arquivo para desabilitar zram
```bash
echo "zram.enable=0" | sudo tee /etc/systemd/zram-generator.conf
```
- Atualizar initramfs (opcional, mas recomendado)
```bash
sudo dracut -f
```
- Reinicar o sistema
```bash
sudo reboot
```
- Configurar o Firewall
```bash
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10252/tcp
sudo firewall-cmd --permanent --add-port=10255/tcp
sudo firewall-cmd --reload
```
- Habilitar o módulo br_netfilter
```bash
sudo modprobe br_netfilter
```
Persistir a configuração:

```bash
echo 'br_netfilter' | sudo tee /etc/modules-load.d/k8s.conf
```

Configurar os parâmetros do kernel:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```
---
## Parte 1: Instalação do containerd
- Fedora
```bash
sudo dnf install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl enable --now containerd
```

- Debian/Ubuntu
```bash
sudo apt update && sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl enable --now containerd
```

- Arch /Manjaro
```bash
sudo pacman -Syu containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl enable --now containerd
```

- openSUSE
```bash
sudo zypper refresh
sudo zypper update -y
sudo zypper install -y conntrack-tools curl wget vim git
sudo zypper addrepo -f https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/openSUSE_Tumbleweed/ containers
sudo zypper addrepo -f https://download.opensuse.org/repositories/devel:/kubic:/kubernetes/openSUSE_Tumbleweed/ kubernetes
sudo zypper refresh
sudo zypper install -y kubernetes-kubeadm kubernetes-kubelet kubernetes-client
sudo systemctl enable --now kubelet
sudo zypper install -y docker
sudo systemctl enable --now docker
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
sudo zypper install -y opennebula-server opennebula-sunstone
sudo systemctl enable --now opennebula opennebula-sunstone
```
## Parte 2: Instalação do Kubernetes (kubeadm, kubelet, kubectl)
- Fedora (via repositório Kubernetes)
```bash
sudo tee /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
EOF

sudo dnf install -y kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

- Debian/Ubuntu
```bash
sudo apt update && sudo apt install -y apt-transport-https curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

- Arch/Manjaro
```bash
sudo pacman -Syu kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

## Parte 3: Inicializar o cluster Kubernetes
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```
- Configure o acesso local para o usuário atual
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Parte 4: Instalar rede de pods (ex.: Calico)
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

## Parte 5: Permitir agendamento de pods no nó master (para testes)

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### Parte 6: Comandos Kubernetes úteis
- Consultar nós e pods
```bash
kubectl get nodes
kubectl get pods -A
```
- Obter detalhes de um pod específico
```bash

kubectl describe pod <nome_do_pod> -n <namespace>

```
- Exemplo
```bash
kubectl describe pod kubernetes-scheduler-fedora -n kube-system
```
Erros como "NotFound" ocorrem quando o nome do pod ou namespace está errado

## Parte 7: Instalação Avançada com Kubespray (Cluster Multi-Nó ou HA)
- Para ambientes mais complexos ou em produção, o Kubespray fornece um conjunto completo de playbooks Ansible para provisionamento de clusters Kubernetes.
### Pré-Requisitos
- Python 3.6+
- Ansible 2.12+
- Acesso SSH sem senha entre os nós (recomenda-se configurar chaves SSH; se não, será necessário digitar a senha manualmente para cada nó).

### Passos para usar o Kubespray
```bash
# Clone o repositório
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray

# Instale as dependências
sudo pip3 install -r requirements.txt

# Copie o inventário de exemplo
cp -rfp inventory/sample inventory/mycluster

# Edite o inventário com IPs dos nós
vim inventory/mycluster/hosts.yaml

# Execute o playbook (como root)
ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root cluster.yml
```

## Parte 8: Testando Pods e Namespaces
- Você pode testar seu cluster criando Pods e trabalhando com Namespaces
```bash
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.14.2
      ports:
        - containerPort: 80
```
```bash
kubectl apply -f pod.yaml
kubectl get pods
```
- Trabalhando com Namespaces
```bash
kubectl create namespace teste
kubectl get namespaces
kubectl get pods -n teste
kubectl config set-context --current --namespace=teste
```
