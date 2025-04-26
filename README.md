# Instalação e Configuração do Kubernetes com Kubeadm e Kubespray

Este documento descreve os passos para **instalar e configurar o Kubernetes** em um cluster utilizando o **kubeadm** para a instalação inicial e o **Kubespray** para configurações mais avançadas, como clusters com múltiplos nós e alta disponibilidade.

---

## 🚀 Pré-requisitos

- Sistema operacional Linux (Ubuntu, Debian, Fedora, CentOS, etc.)
- Acesso root ou sudo aos nós
- SSH configurado para acesso sem senha entre os nós
- Python 3.6+ e Ansible 2.12+ (necessários para o Kubespray)
- Desativar o **swap** em todos os nós:

  ```bash
  sudo swapoff -a
  sudo sed -i '/ swap / s/^/#/' /etc/fstab
  ```

## Instalação Kubernetes com kubeadm
### Debian/Ubuntu
1. Atualize o sistema
```bash
sudo apt update && sudo apt upgrade -y
```
2. Instale as dependências
```bash
sudo apt install -y apt-transport-https ca-certificates curl
```
3. Adicione a chave GPG e o repositório do Kubernetes
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/trusted.gpg.d/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
4. Instale o Kubernetes
```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Fedora
1. Atualize o sistema
```bash
sudo dnf5 update -y
```
2. Adicione o repositório
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
EOF
```
3. Instale o Kubernetes
```bash
sudo dnf5 install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```
### Configuração do Cluster com kubeadm
1. Inicialize o cluster Kubernetes (no nó master)
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
2. Configure o kubectl no nó master:
```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
3. Instale o CNI (Calico ou Flannel). Exemplo com Flannel:
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
4. Adicione nós worker:
- Execute o comando fornecido ao final da inicialização do master em cada nó worker:
```bash
sudo kubeadm join <IP_MASTER>:<PORTA> --token <TOKEN> --discovery-token-ca-cert-hash sha25
```
### Configuração Avançada com Kubespray
1. Clone o repositório do Kubespray:
```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
```
2. Instale as depedências do Python
```bash
sudo pip3 install -r requirements.txt
```
3. Copie o inventário de exemplo
```bash
cp -rfp inventory/sample inventory/mycluster
```
4. Edite o inventário com os IPs dos seus nós
```bash
vim ~/inventory/mycluster/hosts.yaml
```
5. Execute o playbook do Ansible
```bash
ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root cluster.yml
```
## ✅ Verificando o estado do cluster

Após a instalação, use os seguintes comandos para verificar os nós e os pods:

### Verificar os nós do cluster:

```bash
kubectl get nodes
kubectl get pods -A
```