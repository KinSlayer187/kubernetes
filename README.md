# Instalação do Kubernetes com kubeadm no Fedora

Este documento descreve como configurar um cluster Kubernetes usando `kubeadm` no Fedora, incluindo a instalação do `containerd` e a configuração de rede com Calico.

## Pré-requisitos

- Fedora Linux
- Acesso root ou `sudo`
- `kubeadm`, `kubectl`, `kubelet` instalados
- `containerd` instalado

## Etapas

### 1. Instale o containerd

```bash
sudo dnf install -y containerd
```

### 2. Configure o containerd

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

### 3. Habilite e inicie o containerd

```bash
sudo systemctl enable --now containerd
```

### 4. Desabilite o swap (ou configure corretamente para cgroup v2)

```bash
sudo swapoff -a
```

Edite `/etc/fstab` para comentar a linha de swap se necessário.

### 5. Inicialize o cluster com kubeadm

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

### 6. Configure o kubectl para o usuário atual

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 7. Instale o Calico (rede de pods)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

### 8. Remova o taint do nó master para que ele agende pods

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### 9. Verifique o estado do cluster

```bash
kubectl get nodes
kubectl get pods -A
```

## Dicas

- Certifique-se de que as portas 6443 e 10250 estão abertas se o firewalld estiver ativo.
- Use `systemctl enable` para garantir que `containerd` e `kubelet` iniciem automaticamente.

---

Com isso, você terá um cluster Kubernetes funcional no Fedora.
