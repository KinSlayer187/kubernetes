
Este guia apresenta um **passo a passo estruturado** para diagnóstico e resolução de problemas em clusters Kubernetes, com **comandos práticos**, foco em **rede (Calico + MetalLB)**, **Ingress NGINX + TLS**, **Storage (PVC/PV/StorageClass)** e uma **tabela de sintomas, causas e soluções**.

---

## 1️⃣ Metodologia Passo a Passo

### Passo 1 — Identificar o escopo

* Aplicação, namespace, nó específico ou cluster inteiro?
* Falha pontual ou recorrente?

```bash
kubectl get nodes
kubectl get pods -A
```

---

### Passo 2 — Verificar eventos

Eventos geralmente indicam a causa raiz do problema.

```bash
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

---

### Passo 3 — Inspecionar o recurso afetado

```bash
kubectl describe pod <pod> -n <namespace>
kubectl describe node <node>
kubectl describe svc <service> -n <namespace>
```

---

### Passo 4 — Analisar logs

```bash
kubectl logs <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous
```

---

### Passo 5 — Validar dependências

* Rede (CNI / MetalLB)
* Storage
* Registry (Harbor)
* Ingress

---

## 2️⃣ Troubleshooting de Rede (Calico + MetalLB)

### Sintomas comuns

* Pods sem comunicação
* Service LoadBalancer sem IP externo
* Timeout ou perda de pacotes

### Comandos úteis

```bash
kubectl get pods -n calico-system
kubectl get ippools
kubectl get bgpconfigurations
```

Verificar MetalLB:

```bash
kubectl get pods -n metallb-system
kubectl get svc -A | grep LoadBalancer
```

### Causas comuns

* CNI não instalado corretamente
* IP Pool inválido ou conflitante
* Rede física bloqueando ARP/BGP

---

## 3️⃣ Troubleshooting de Ingress NGINX + TLS

### Sintomas comuns

* Erro 404, 502 ou 504
* HTTPS não funciona
* Certificado inválido

### Comandos úteis

```bash
kubectl get ingress -A
kubectl describe ingress <ingress> -n <namespace>
```

Logs do Ingress Controller:

```bash
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

Verificar certificados:

```bash
kubectl get secret -n <namespace>
```

### Causas comuns

* Service ou porta incorreta
* Secret TLS inexistente
* cert-manager não funcional

---

## 4️⃣ Troubleshooting de Storage (PVC, PV, StorageClass)

### Sintomas comuns

* PVC em `Pending`
* Pod não inicia por volume
* Erro de permissão

### Comandos úteis

```bash
kubectl get pv
kubectl get pvc -A
kubectl get storageclass
```

Inspecionar PVC:

```bash
kubectl describe pvc <pvc> -n <namespace>
```

### Causas comuns

* StorageClass inexistente ou incorreta
* Provisioner não instalado
* Falta de espaço no backend

---

## 5️⃣ Troubleshooting de Pods

### Pods em Pending

```bash
kubectl describe pod <pod> -n <namespace>
```

Causas:

* Falta de recursos
* PVC pendente

---

### Pods em CrashLoopBackOff

```bash
kubectl logs <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous
```

Causas:

* Erro na aplicação
* ConfigMap ou Secret incorreto

---

## 6️⃣ Tabela — Sintoma → Causa → Solução

| Sintoma          | Causa Provável                | Solução                        |
| ---------------- | ----------------------------- | ------------------------------ |
| Node NotReady    | CNI ausente ou kubelet falhou | Verificar CNI e kubelet        |
| Pod Pending      | Falta de recursos ou PVC      | Ajustar requests ou storage    |
| ImagePullBackOff | Credencial inválida           | Corrigir imagePullSecrets      |
| Service sem IP   | MetalLB mal configurado       | Ajustar IPAddressPool          |
| Ingress 404      | Path ou Service incorreto     | Revisar regras do Ingress      |
| TLS inválido     | Secret inexistente            | Recriar certificado            |
| PVC Pending      | StorageClass inválida         | Criar ou corrigir StorageClass |

---

## 7️⃣ Boas Práticas

* Monitorar continuamente o cluster
* Centralizar logs e métricas
* Usar namespaces e NetworkPolicies
* Documentar incidentes

---

## Conclusão

Um processo estruturado de troubleshooting reduz drasticamente o tempo de indisponibilidade e aumenta a confiabilidade do cluster Kubernetes em ambientes de produção.
