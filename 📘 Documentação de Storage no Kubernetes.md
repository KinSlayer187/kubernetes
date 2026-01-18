## 1. Contexto do Ambiente

Este ambiente Kubernetes foi montado para estudos e práticas reais de produção, envolvendo:

- Cluster Kubernetes (1 master + 1 worker)
- Container Runtime: containerd
- Ingress NGINX
- Cert-Manager
- Harbor
- Storage inicial com **local-path**
- Migração para **Longhorn**
- Workloads críticos convertidos para **StatefulSet**

O objetivo principal foi **garantir persistência real de dados**, inclusive após **reboot do worker**, e documentar todos os erros reais encontrados no processo.

---

## 2. Storage Inicial – local-path

### 2.1 O que é o local-path

O `local-path` é um provisionador simples que cria volumes locais no filesystem do node onde o Pod roda.

Características:

- Apenas para **testes/labs**
- Sem replicação
- Sem failover
- Dados perdidos se o node cair

### 2.2 PVC usando local-path

Exemplo de PVC criado inicialmente:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi
```

Verificação:

```bash
kubectl get pvc
kubectl get pvc -o wide
```

Resultado esperado:

- STATUS: Bound
- STORAGECLASS: local-path

### 2.3 Limitação encontrada

Apesar de funcionar, o `local-path`:

- Não permite reboot seguro
- Não permite migração de Pod
- Não é aceitável para banco de dados

Isso motivou a migração para Longhorn.

---

## 3. Introdução ao Longhorn

### 3.1 O que é o Longhorn

Longhorn é um **storage distribuído para Kubernetes**, oferecendo:

- Volumes replicados
- Persistência real
- Snapshots
- Backups
- Integração CSI

### 3.2 Instalação do Longhorn

(Longhorn já estava instalado previamente no master e worker)

Componentes verificados:

```bash
kubectl -n longhorn-system get pods
```

Todos os pods essenciais:

- longhorn-manager
- instance-manager
- csi-* (attacher, provisioner, resizer, snapshotter)

---

## 4. Problema Crítico: Espaço em Disco

### 4.1 Erro encontrado

Volumes Longhorn ficaram em estado:

- `faulted`
- `detached`
- `ReplicaSchedulingFailure`

Mensagem chave:

> insufficient storage; precheck new replica failed

### 4.2 Diagnóstico

No worker:

```bash
df -h /var
lsblk
```

Descoberto:

- `/var` tinha apenas ~14GB
- Longhorn estava usando `/var/lib/longhorn`

### 4.3 Solução: mover Longhorn para /srv

Disco grande disponível em:

- `/srv` (~500GB)

Configuração feita:

1. Desabilitar disco padrão:

```bash
kubectl -n longhorn-system patch nodes.longhorn.io worker \
  --type=merge \
  -p '{"spec":{"disks":{"default-disk":{"allowScheduling":false}}}}'
```

2. Adicionar novo disco:

```bash
kubectl -n longhorn-system patch nodes.longhorn.io worker \
  --type=merge \
  -p '{
    "spec": {
      "disks": {
        "srv-disk": {
          "path": "/srv/longhorn",
          "allowScheduling": true,
          "storageReserved": 0
        }
      }
    }
  }'
```

Verificação:

```bash
kubectl -n longhorn-system describe nodes.longhorn.io worker
```

---

## 5. Introdução ao StatefulSet

### 5.1 Por que StatefulSet

StatefulSet é obrigatório para workloads que exigem:

- Identidade estável
- PVC fixo por réplica
- Persistência real

Exemplos:

- PostgreSQL
- MySQL
- Redis
- GitLab
- Harbor

---

## 6. StatefulSet com PostgreSQL

### 6.1 Primeiro erro real encontrado (lost+found)

Logs do Pod:

```bash
kubectl logs example-db-0
```

Erro:

> directory "/var/lib/postgresql/data" exists but is not empty  
> It contains a lost+found directory

### 6.2 Causa

- Longhorn monta o volume como filesystem
- O diretório raiz contém `lost+found`
- PostgreSQL não aceita isso

### 6.3 Correção correta

Usar subdiretório com variável `PGDATA`.

---

## 7. StatefulSet Final (FUNCIONAL)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: example-db
spec:
  serviceName: example-db
  replicas: 1
  selector:
    matchLabels:
      app: example-db
  template:
    metadata:
      labels:
        app: example-db
    spec:
      containers:
      - name: db
        image: postgres:16
        env:
        - name: POSTGRES_PASSWORD
          value: example
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: longhorn
      resources:
        requests:
          storage: 2Gi
```

Aplicação correta:

```bash
kubectl apply -f StatefulSet.yaml
```

---

## 8. Verificações Importantes

### 8.1 Pods

```bash
kubectl get pods
```

Resultado esperado:

- example-db-0 → Running

### 8.2 PVC

```bash
kubectl get pvc
```

- STORAGECLASS: longhorn

### 8.3 Volumes Longhorn

```bash
kubectl -n longhorn-system get volumes.longhorn.io
```

Estado esperado em lab:

- attached
- degraded (normal com 1 worker)

---

## 9. Observação Importante: Estado Degraded

Com apenas **1 worker**:

- Réplicas configuradas > 1
- Longhorn mostra `degraded`

Isso **não é erro** em ambiente de laboratório.

Em produção:

- Usar ≥ 3 workers
- Réplicas = 3

---

## 10. Conclusão

Este ambiente agora possui:

- Storage persistente real
- StatefulSet correto
- PostgreSQL funcional
- Longhorn usando disco adequado
- Base sólida para Harbor, GitLab, etc.

Esta documentação reflete **erros reais e soluções reais**, sendo adequada para:

- Estudo avançado
- Ambientes de teste
- Base de produção
- Repositórios técnicos