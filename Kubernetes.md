## Visão geral

**Kubernetes (K8s)** é uma plataforma de orquestração de contêineres open source que automatiza a **implantação**, **escala**, **gerenciamento** e **alta disponibilidade** de aplicações conteinerizadas. Ele abstrai a infraestrutura subjacente (on‑premises, cloud ou híbrida) e fornece mecanismos para balanceamento de carga, auto‑recuperação, atualizações contínuas (rolling updates), gerenciamento de configuração e segredos.

Com Kubernetes, aplicações são executadas em **clusters** compostos por nós (nodes), organizadas em **Pods**, e controladas por recursos declarativos como **Deployments**, **Services** e **Ingress**, permitindo operações consistentes, escaláveis e resilientes em produção.

## Arquitetura do Cluster Kubernetes

Um **cluster Kubernetes** é composto por um conjunto de máquinas (físicas ou virtuais) organizadas em dois grandes grupos: **Control Plane** e **Worker Nodes**. Essa arquitetura separa claramente o plano de controle do plano de execução das aplicações.

### Control Plane (Plano de Controle)

Responsável por **gerenciar o cluster** e manter o estado desejado dos recursos.

Componentes principais:

* **kube-apiserver**: Interface central do cluster. Recebe e valida todas as requisições (kubectl, APIs, controllers).
* **etcd**: Banco de dados distribuído que armazena o estado do cluster (configurações, segredos, objetos).
* **kube-scheduler**: Decide em qual Worker Node cada Pod será executado, considerando recursos e políticas.
* **kube-controller-manager**: Executa controladores que garantem o estado desejado (replicas, nós, endpoints).
* **cloud-controller-manager** (opcional): Integra o Kubernetes a provedores de nuvem (load balancer, volumes, IPs).

### Worker Nodes (Nós de Trabalho)

Responsáveis por **executar as aplicações**.

Componentes principais:

* **kubelet**: Agente que garante que os Pods definidos no cluster estejam rodando corretamente no nó.
* **Container Runtime**: Responsável por executar os contêineres (ex: **containerd**, **CRI-O**).
* **kube-proxy**: Gerencia a comunicação de rede entre Pods e Services (iptables/IPVS).

### Comunicação e Fluxo

1. O usuário aplica manifestos via `kubectl`.
2. O **kube-apiserver** valida e registra o estado no **etcd**.
3. O **scheduler** seleciona o nó adequado.
4. O **kubelet** cria os Pods no Worker Node.
5. O **kube-proxy** garante o acesso de rede aos Services.

### Visão Geral da Arquitetura

* Arquitetura **distribuída e escalável**
* Separação entre **gerenciamento** e **execução**
* Suporte a **alta disponibilidade** do Control Plane
* Funciona em ambientes **on-premises, cloud e híbridos**

Essa arquitetura permite que o Kubernetes mantenha aplicações **resilientes, escaláveis e autogerenciáveis**, mesmo em ambientes complexos de produção.
## Instalação do Kubernetes (Visão Geral por Distribuição)

A instalação do Kubernetes pode variar conforme a distribuição Linux, mas o fluxo geral é semelhante em todas elas. Esta seção apresenta uma **visão geral da instalação** no **Debian**, **Ubuntu**, **Fedora** e **openSUSE**, focando nos pontos comuns e nas diferenças principais.

### Etapas Comuns a Todas as Distribuições

Independentemente da distribuição, a instalação do Kubernetes segue estes passos:

1. **Preparação do sistema**
   * Desativar swap
   * Ajustar parâmetros de kernel (bridge, iptables, forwarding)
   * Sincronizar data e hora
2. **Instalação do runtime de contêiner**
   * Recomendado: **containerd** ou **CRI-O**
3. **Instalação dos componentes do Kubernetes**
   * `kubeadm`: inicialização do cluster
   * `kubelet`: agente do nó
   * `kubectl`: ferramenta de gerenciamento

4. **Inicialização do cluster**
   * Executada no nó Control Plane com `kubeadm init`

5. **Adição de Worker Nodes**
   * Realizada com `kubeadm join`

6. **Instalação de um plugin de rede (CNI)**
   * Exemplos: Calico, Flannel, Cilium

---

## Debian

* Uso intensivo do **APT** para gerenciamento de pacotes
* Kubernetes instalado via repositório oficial
* Muito utilizado em ambientes **servidor** e **produção**
* Alta estabilidade e suporte a longo prazo

Ideal para ambientes on-premises e clusters estáveis.

---

## Ubuntu

* Baseado no Debian, com maior facilidade inicial
* Amplo suporte da comunidade Kubernetes
* Excelente compatibilidade com ferramentas cloud
* Muito usado em laboratórios e ambientes produtivos

Ideal para quem busca rapidez na implantação.

---

## Fedora

* Distribuição moderna e atualizada
* Uso do **DNF** como gerenciador de pacotes
* Forte integração com **containerd**, **Podman** e **CRI-O**
* Próxima das tecnologias upstream do Kubernetes

Ideal para testes avançados, DevOps e ambientes híbridos.

---

## openSUSE

* Disponível nas variantes **Leap** (estável) e **Tumbleweed** (rolling release)
* Boa integração com ferramentas corporativas
* Gerenciamento via **Zypper**
* Bastante utilizada em ambientes empresariais

Ideal para ambientes corporativos e infraestrutura tradicional.

---

### Considerações Importantes

* A versão do Kubernetes deve ser **compatível** entre todos os nós
* O runtime de contêiner deve estar configurado antes do kubeadm
* Plugins de rede são obrigatórios para o funcionamento do cluster
* Para produção, recomenda-se **alta disponibilidade** no Control Plane

Essa abordagem multi-distribuição garante flexibilidade na escolha do sistema operacional sem comprometer a arquitetura e o funcionamento do Kubernetes.
## Rede Calico no Kubernetes

O **Calico** é um dos plugins de rede (**CNI – Container Network Interface**) mais utilizados no Kubernetes. Ele fornece **conectividade de rede**, **políticas de segurança** e **controle de tráfego** entre Pods, Nodes e Services, sendo amplamente adotado em ambientes de produção.

### O que o Calico faz

* Garante comunicação **Pod-to-Pod** entre todos os nós do cluster
* Implementa **Network Policies** (controle de tráfego L3/L4)
* Permite isolamento e segmentação de aplicações
* Funciona em ambientes **on-premises**, **cloud** e **híbridos**

Sem um CNI como o Calico, os Pods não conseguem se comunicar corretamente.

---

## Componentes do Calico

### Felix

* Agente executado em cada nó
* Programa regras de rede no kernel Linux (iptables, nftables ou eBPF)
* Garante a aplicação das políticas de rede

### BGP (Bird)

* Utiliza o protocolo **BGP** para troca de rotas entre os nós
* Permite comunicação eficiente sem encapsulamento (modo padrão)

### Typha (opcional)

* Otimiza clusters grandes
* Reduz a carga de conexões no datastore

### Datastore

* Pode ser o **etcd do Kubernetes** ou o próprio **Kubernetes API**
* Armazena políticas, rotas e estado do Calico

---

## Modos de Funcionamento

### Modo BGP (sem encapsulamento)

* Melhor desempenho
* Requer infraestrutura de rede compatível
* Ideal para data centers e ambientes on-premises

### Modo IP-in-IP / VXLAN

* Utiliza encapsulamento de pacotes
* Funciona em redes que não suportam BGP
* Muito usado em cloud pública

---

## Network Policies

O Calico permite criar **políticas de rede** para controlar o tráfego entre Pods e Namespaces.

Exemplos de controle:

* Permitir ou bloquear comunicação entre aplicações
* Restringir acesso a bancos de dados
* Implementar modelo **Zero Trust** dentro do cluster

As políticas são declarativas e aplicadas dinamicamente.

---

## Vantagens do Calico

* Alto desempenho e baixa latência
* Segurança avançada com Network Policies
* Suporte a clusters grandes
* Compatível com kubeadm, Kubespray e clouds

---

## Considerações Importantes

* Deve ser instalado **após** o `kubeadm init`
* É obrigatório para que os nós fiquem em estado **Ready**
* A escolha do modo (BGP ou encapsulamento) impacta desempenho e compatibilidade

O Calico é uma solução robusta, segura e amplamente validada para redes Kubernetes em ambientes de produção.
## Ingress NGINX no Kubernetes

O **Ingress NGINX** é um controlador de Ingress amplamente utilizado no Kubernetes para **expor aplicações HTTP e HTTPS** para fora do cluster. Ele atua como um **proxy reverso** baseado no NGINX, permitindo o roteamento de tráfego por **host**, **path** e **TLS**, de forma centralizada e escalável.

O recurso **Ingress** define as regras, enquanto o **Ingress Controller (NGINX)** é o componente responsável por aplicá-las.

---

## Componentes Principais

### Ingress Resource

* Objeto Kubernetes que descreve regras de acesso externo
* Define hosts, caminhos (paths) e serviços de destino

### Ingress NGINX Controller

* Executa como Pods no cluster
* Observa objetos Ingress e configura dinamicamente o NGINX
* Suporta balanceamento de carga, TLS, rewrite e autenticação

---

## Funcionamento Geral

1. O cliente faz uma requisição HTTP/HTTPS.
2. O tráfego chega ao **Ingress NGINX** (via Service LoadBalancer ou NodePort).
3. O controlador avalia as regras do **Ingress**.
4. A requisição é encaminhada ao **Service** correto.
5. O Service direciona o tráfego aos **Pods**.

---

## Modos de Exposição

* **LoadBalancer**: Integra-se a provedores de nuvem ou MetalLB (on-premises).
* **NodePort**: Exposição direta pela porta do nó.
* **HostNetwork** (avançado): Usa a rede do host para menor latência.

---

## TLS e HTTPS

O Ingress NGINX suporta **TLS nativo**, permitindo:

* Certificados estáticos (Secrets)
* Integração com **cert-manager** (Let's Encrypt)
* Redirecionamento automático HTTP → HTTPS

---

## Annotations

O Ingress NGINX utiliza **annotations** para ajustes finos, como:

* Rewrite de URLs
* Limite de requisições (rate limit)
* Timeouts
* Autenticação básica ou OAuth

Essas configurações tornam o Ingress altamente flexível.

---

## Vantagens do Ingress NGINX

* Simples de configurar
* Amplamente documentado e testado
* Alta compatibilidade com aplicações web
* Excelente integração com Kubernetes padrão

---

## Considerações Importantes

* O Ingress Controller deve estar instalado antes do uso do recurso Ingress
* Requer um **CNI funcional** (ex: Calico)
* Para produção, recomenda-se TLS + alta disponibilidade

O Ingress NGINX é a solução padrão de mercado para exposição de aplicações web em clusters Kubernetes.
## Harbor no Kubernetes

O **Harbor** é um **registry de contêineres corporativo (enterprise-grade)**, open source, utilizado para armazenar, gerenciar e distribuir imagens de contêiner com foco em **segurança**, **controle de acesso** e **conformidade**. Ele é amplamente usado em ambientes Kubernetes para substituir ou complementar registries públicos.

O Harbor estende um registry Docker padrão adicionando recursos avançados essenciais para produção.

---

## Principais Funcionalidades

* **Armazenamento de imagens** Docker/OCI
* **Controle de acesso baseado em papéis (RBAC)**
* **Autenticação integrada** (LDAP, OIDC, SSO)
* **Escaneamento de vulnerabilidades** (Trivy)
* **Assinatura e verificação de imagens** (Notary/Cosign)
* **Replication** entre registries
* **Interface Web (UI)** e API REST

---

## Arquitetura do Harbor

O Harbor é composto por vários serviços, normalmente executados como Pods no Kubernetes:

* **Harbor Core**: Gerencia projetos, usuários, permissões e políticas
* **Registry**: Serviço de armazenamento de imagens
* **Database (PostgreSQL)**: Armazena metadados
* **Redis**: Cache e filas internas
* **Trivy**: Scanner de vulnerabilidades
* **Job Service**: Execução de tarefas assíncronas (replicação, scan)
* **Portal**: Interface Web

Essa arquitetura modular permite **escalabilidade** e **alta disponibilidade**.

---

## Harbor e Kubernetes

Em ambientes Kubernetes, o Harbor é utilizado para:

* Servir como **registry privado** do cluster
* Centralizar imagens internas da organização
* Aplicar políticas de segurança antes do deploy
* Integrar pipelines CI/CD (GitLab CI, Jenkins, GitHub Actions)

O Kubernetes consome imagens do Harbor através de **Secrets de registry** (`imagePullSecrets`).

---

## Segurança de Imagens

O Harbor oferece recursos avançados de segurança:

* **Vulnerability Scanning** automático
* Bloqueio de deploy de imagens vulneráveis
* Assinatura e verificação de imagens confiáveis
* Políticas de retenção e limpeza de imagens

Esses recursos ajudam a manter um **supply chain seguro**.

---

## Modos de Implantação

* **Standalone** (Docker Compose)
* **Kubernetes** (Helm Chart oficial)
* **Alta disponibilidade (HA)** para produção

Para clusters Kubernetes, o uso via **Helm** é o método recomendado.

---

## Vantagens do Harbor

* Total controle sobre imagens
* Segurança avançada integrada
* Ideal para ambientes corporativos e regulados
* Excelente integração com Kubernetes

---

## Considerações Importantes

* Requer armazenamento persistente confiável
* TLS é altamente recomendado em produção
* Deve ser integrado ao fluxo CI/CD

O Harbor é uma peça fundamental em arquiteturas Kubernetes modernas que exigem **segurança, governança e controle total** sobre imagens de contêiner.
## Troubleshooting no Kubernetes

O **troubleshooting no Kubernetes** envolve identificar, analisar e corrigir problemas relacionados a **Pods**, **Nodes**, **rede**, **storage**, **Ingress**, **control plane** e integrações externas. Uma abordagem estruturada é essencial para reduzir o tempo de indisponibilidade em ambientes de produção.

---

## Abordagem Geral de Diagnóstico

1. **Identificar o escopo do problema**

   * Aplicação específica ou cluster inteiro?
   * Falha intermitente ou permanente?

2. **Verificar o estado dos recursos**

   * Nodes, Pods, Services e Ingress

3. **Analisar eventos e logs**

   * Eventos do Kubernetes
   * Logs de Pods e componentes do sistema

4. **Validar dependências**

   * Rede (CNI)
   * Storage
   * Registry (Harbor)

---

## Problemas Comuns e Diagnóstico

### Nodes em estado `NotReady`

Causas comuns:

* Falha no runtime de contêiner
* Problemas de rede (CNI não funcional)
* Falta de recursos (CPU, memória, disco)

Verificações típicas:

* Status do kubelet
* Logs do runtime
* Comunicação com o Control Plane

---

### Pods em `Pending`

Causas comuns:

* Falta de recursos no nó
* Problemas com StorageClass ou PVC
* Scheduler não consegue alocar o Pod

Analisar:

* Eventos do Pod
* Requests e Limits de recursos

---

### Pods em `CrashLoopBackOff`

Causas comuns:

* Erro na aplicação
* Variáveis de ambiente incorretas
* Problemas de configuração (ConfigMap / Secret)

Ações:

* Analisar logs do contêiner
* Verificar comandos e argumentos

---

### Problemas de Rede (Calico)

Sintomas:

* Pods não se comunicam
* Services inacessíveis

Causas comuns:

* Calico não instalado corretamente
* NetworkPolicies bloqueando tráfego
* Problemas de encapsulamento ou BGP

---

### Problemas com Ingress NGINX

Sintomas:

* Erro 404 / 502 / 504
* TLS não funciona

Causas comuns:

* Ingress mal configurado
* Service inexistente ou porta incorreta
* Certificados TLS inválidos

---

### Erros ao puxar imagens (Harbor)

Sintomas:

* `ImagePullBackOff`
* `ErrImagePull`

Causas comuns:

* Credenciais inválidas
* TLS não confiável
* Imagem inexistente

---

## Ferramentas Essenciais

* `kubectl get` / `describe`
* `kubectl logs`
* `kubectl exec`
* Eventos do Kubernetes
* Logs do kubelet e container runtime

---

## Boas Práticas de Troubleshooting

* Monitorar o cluster continuamente
* Utilizar logs centralizados
* Implementar alertas proativos
* Documentar incidentes e soluções

---

## Considerações Finais

Um bom processo de troubleshooting no Kubernetes reduz significativamente o tempo de resolução de incidentes e aumenta a confiabilidade do ambiente. Conhecer a arquitetura, a rede, o Ingress e as integrações é fundamental para diagnósticos eficientes.
