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
## Criando um Pod manualmente
- Após a instalação e configuração do cluster, você pode testar se tudo está funcionando corretamente criando um pod manualmente
1. Crie um arquivo pod.yaml com o seguinte conteúdo:
```yaml
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
2. Aplique o pod no cluster
```bash
kubectl apply -f pod.yaml
```
3. Verifique se o pod foi criado com sucesso
```bash
kubectl get pods
```
## Instalação do krew (gerenciador de plugins para kubectl)
- O krew permite buscar, instalar e gerenciar plugins extras para o kubectl
1. Instalar o krew
```bash
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m)" &&
  # Corrige nomes de arquitetura
  if [ "$ARCH" = "x86_64" ]; then ARCH="amd64"; fi
  if [ "$ARCH" = "aarch64" ]; then ARCH="arm64"; fi
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
```
2. Após isso, adicione o diretório do Krew no seu PATH
```bash
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```
3. Verifique a instalação do Krew
```bash
kubectl krew version
```
3. Exemplo de uso
- Buscar plugins disponíveis
```bash
kubectl krew search
```
- Instalar um plugin
```bash
kubectl krew install ctx
```
- Listar plugins instalados
```bash
kubectl krew list
```

## Trabalhando com namespace no Kubernetes
Namespaces são usados para ogranizar e isolar recursos dentro do cluster Kubernetes
### Criando um namespace
```bash
kubectl create namespace meu-namespace
```
1. Usando um Namespace num YAML (Exemplo atualizado do Pod)
- Você pode utilizar diretamente o namespace dentro do YAML do recurso, como no exemplo abaixo
```bash
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: meu-namespace
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```
- Ou pode aplicar o YAML e depois mover o recurso para o namespace desejado (não é o ideal)
2. Listar os Namespaces
```bash
kubectl get namespaces
```
3. Verifica recursos dentro de um Namespace específico
```bash
kubectl get pods -n meu-namespace
```
3. Definir um Namespace padrão para o kubectl
- Se quiser trabalhar dentro de um namespace sem precisar passar -n, configure
```bash
kubectl config set-context --current --namespace=meu-namespace (não é a forma mais ideal, mas ficará para conhecimento)
```
