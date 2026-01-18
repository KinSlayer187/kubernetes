
Este documento consolida a configuração correta do **Ingress NGINX**, **Harbor Registry** e **Podman**, incluindo resolução de erros comuns, push/pull de imagens e uso de **Secrets** no Kubernetes.

---

## 1. Ingress NGINX – Controller

### 1.1 Deployment

O **controller NÃO substitui `ports`**.

Os ports **80 e 443 devem permanecer**, pois são usados pelo Ingress para HTTP/HTTPS.

Exemplo (trecho relevante):

```yaml
containers:
- name: controller
  ports:
  - containerPort: 80
    name: http
    protocol: TCP
  - containerPort: 443
    name: https
    protocol: TCP
```

> ⚠️ Nunca remova os ports do controller do ingress-nginx.

---

## 2. Service do Ingress NGINX

Exemplo usando **NodePort**:

```bash
kubectl get svc -n ingress-nginx
```

Saída esperada:

```text
ingress-nginx-controller   NodePort   10.x.x.x   <none>   80:31856/TCP,443:30831/TCP
```

➡️ Acesso externo:

- HTTP: `http://IP_DO_NODE:31856`
    
- HTTPS: `https://IP_DO_NODE:30831`
    

---

## 3. Harbor – Validação de Acesso

Teste direto no registry:

```bash
curl -k -I https://harbor.local:30831/v2/
```

Resposta correta:

```text
HTTP/2 401
Docker-Distribution-API-Version: registry/2.0
```

> ✅ O código **401 é esperado** (registry exige autenticação).

---

## 4. Podman – Pull, Tag e Push (Forma Correta)

### 4.1 Erro comum

```text
short-name "nginx:alpine" did not resolve
```

### 4.2 Pull correto (nome totalmente qualificado)

```bash
podman pull docker.io/library/nginx:alpine
```

### 4.3 Tag para o Harbor

```bash
podman tag docker.io/library/nginx:alpine harbor.local:30831/library/nginx:1.0
```

### 4.4 Push para o Harbor

```bash
podman push harbor.local:30831/library/nginx:1.0
```

---

## 5. (Opcional) Permitir nomes curtos no Podman

> ⚠️ **Não recomendado em produção**

```bash
sudo nano /etc/containers/registries.conf
```

```toml
unqualified-search-registries = ["docker.io"]
```

---

## 6. Secret do Harbor no Kubernetes

### 6.1 Criar Secret

```bash
kubectl create secret docker-registry harbor-regcred \
  --docker-server=harbor.local:30831 \
  --docker-username=USUARIO \
  --docker-password=SENHA \
  --docker-email=email@local \
  -n default
```

### 6.2 Alterar username / password / email

#### Opção recomendada – Recriar

```bash
kubectl delete secret harbor-regcred -n default
```

```bash
kubectl create secret docker-registry harbor-regcred \
  --docker-server=harbor.local:30831 \
  --docker-username=NOVO_USUARIO \
  --docker-password=NOVA_SENHA \
  --docker-email=novo@email.local \
  -n default
```

> ✔ Deployments passam a usar o novo Secret automaticamente.

---

## 7. Uso do Secret no Deployment

```yaml
spec:
  imagePullSecrets:
  - name: harbor-regcred
  containers:
  - name: nginx
    image: harbor.local:30831/library/nginx:1.0
```

---

## 8. Troubleshooting Rápido

### Erro: `Could not resolve host harbor.local`

**Causa:** DNS ou `/etc/hosts` não configurado.

**Solução:**

```bash
sudo nano /etc/hosts
```

```text
IP_DO_NODE harbor.local
```

---

### Erro: `connection refused` na porta 443

**Causa:** Tentativa de acesso direto ao Harbor sem passar pelo NodePort do Ingress.

**Solução:**

- Em bare metal, use sempre:
    
    ```text
    https://harbor.local:30831
    ```
    
- A porta 443 só funciona diretamente se houver LoadBalancer.
    

---

### Erro: `x509: certificate signed by unknown authority`

**Causa:** Certificado self-signed não confiável no host.

**Solução (Debian/Ubuntu):**

```bash
sudo cp ca.crt /usr/local/share/ca-certificates/harbor-ca.crt
sudo update-ca-certificates
```

---

### Erro: `server gave HTTP response to HTTPS client`

**Causa:** Tentativa de HTTPS em serviço exposto apenas em HTTP.

**Solução:**

- Verifique se o Harbor está com `externalURL` usando `https://`
    
- Confirme TLS habilitado no `harbor-values.yaml`
    

---

### Erro: `provided port is already allocated`

**Causa:** NodePort em conflito com Ingress NGINX.

**Solução:**

- Remover NodePort do Harbor
    
- Manter NodePort **somente no Ingress Controller**
    

---

### Pod em `ImagePullBackOff`

**Causa:** Secret ausente ou incorreto.

**Solução:**

```bash
kubectl describe pod POD_NAME
kubectl get secret harbor-regcred
```

---

## 9. Boas Práticas

✔ Sempre usar nomes de imagem totalmente qualificados  
✔ Manter TLS no Harbor  
✔ Usar Secrets por namespace  
✔ Não editar secrets manualmente em base64  
✔ Ingress separado do registry

---

📌 **Status atual:**

- Ingress NGINX funcionando
    
- Harbor acessível via NodePort
    
- Podman autenticando corretamente
    
- Push/Pull operacional
    

---

# 10. Migração para LoadBalancer com MetalLB

## 10.1 Instalação do MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

Criar pool de IPs (ajuste para sua rede):

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.240-192.168.0.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2
  namespace: metallb-system
```

```bash
kubectl apply -f metallb-pool.yaml
```

## 10.2 Alterar Service do Ingress para LoadBalancer

```bash
kubectl patch svc ingress-nginx-controller -n ingress-nginx \
  -p '{"spec": {"type": "LoadBalancer"}}'
```

Validar:

```bash
kubectl get svc -n ingress-nginx
```

Agora o Ingress terá **IP externo real**.

---

# 11. Ingress + TLS Automático (cert-manager)

## 11.1 Instalar cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml
```

## 11.2 Criar ClusterIssuer (Let's Encrypt – Staging)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: admin@local
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

```bash
kubectl apply -f clusterissuer.yaml
```

## 11.3 Harbor com TLS Automático

Trecho do `harbor-values.yaml`:

```yaml
expose:
  type: ingress
  tls:
    enabled: true
    certSource: auto
  ingress:
    hosts:
      core: harbor.local
    annotations:
      kubernetes.io/ingress.class: nginx
      cert-manager.io/cluster-issuer: letsencrypt-staging

externalURL: https://harbor.local
```

```bash
helm upgrade harbor harbor/harbor -n harbor -f harbor-values.yaml
```

Validar:

```bash
kubectl get certificate -n harbor
```

---

# 12. Harbor + Kubernetes Scanning

## 12.1 Ativar Scanning no Harbor

- Login no Harbor UI
    
- Administration → Interrogation Services
    
- Trivy: **Enabled**
    

## 12.2 Criar Policy de Scan Automático

- Administration → Vulnerability
    
- Ativar **Scan on Push**
    
- Definir severidade mínima (High / Critical)
    

## 12.3 Verificar Scan

```bash
podman push harbor.local/library/nginx:1.0
```

- A imagem será analisada automaticamente
    
- Resultados visíveis na UI do Harbor
    

## 12.4 Bloquear Deploy (Opcional)

- Criar **Project Policy**
    
- Bloquear imagens com vulnerabilidades críticas
    

---

📌 **Status final:**

- Ingress com IP real (MetalLB)
    
- Harbor exposto via LoadBalancer
    
- TLS automático com cert-manager
    
- Podman e Kubernetes confiáveis
    
- Scanning automático ativo
    
- Pronto para produção bare metal