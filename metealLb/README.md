# MetalLB
Um recurso de rede onde seu papel é atribuir um IP da minha rede local para o service: Loadbalancer, usando dois custom resources - IPAddressPool, define o range de IP disponnivel e um L2Advertisement, que faz o IP ser anunciado via ARP na rede local.

>Em Cloud isso não é necessário, porque quando você expõe um service do tipo LoadBalancer, o próprio provedor provisiona automaticamente o IP.

Em bare-metal isso não existe, sem o o MetalLB o service fica com o IP "pending" pra sempre.

## How-To instalação com 'kubectl apply'

- **1. Instalar**
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml

# Acompanhar a subida dos pods
kubectl get pods -n metallb-system --watch
```
> **Nota**: é recomendado escolher um range de IP na sua rede para que não tenha conflito na distribuição.

- **Após definir o range de IP, podemos seguir com a criação dos CRDs**

- **2. criar um arquivo yaml** *metallb-config.yaml*
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.XXX-192.168.0.XXX # Range de IP escolhido
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - homelab-pool # Nome definido para o IPAddressPool
```
- **3. Para validar, podemos criar um service do tipo LoadBalancer de teste**
```bash
kubectl expose deployment traefik -n kube-system --name=traefik-test --port=80 --type=LoadBalance

# Retorno esperado é que o `EXTERNAL-IP` saiu da faixa configurada (não ficou `<pending>`) e em seguinda podemos deletar.

kubectl delete svc traefik-test -n kube-system
```

## How-To instalação com Terraform

- **1. Criação do arquivo .tf**
```nginx
# Faz a instalação do MetalLB no cluster Kubernetes usando o Helm
resource "helm_release" "metallb" {
  name       = "metallb"
  repository = "https://metallb.github.io/metallb"
  chart      = "metallb"
  version    = "0.16.1"
  namespace  = "metallb-system"
  create_namespace = true

# Aguarda os CRDs e Webhooks estarem prontos antes de concluir
  wait             = true
}

# Pausa para o Webhook do MetalLB estabilizar completamente
resource "time_sleep" "wait_for_webhook" {
  depends_on = [helm_release.metallb]
  create_duration = "30s"
}

# Cria o recurso IPAddressPool do MetalLB
resource "kubectl_manifest" "metallb_IPAddressPool" {
    depends_on = [time_sleep.wait_for_webhook] # Aguarda o Webhook do MetalLB estabilizar antes de criar o recurso IPAddressPool
    yaml_body = yamlencode({ # yamlencode converte o bloco de código em formato YAML para ser usado pelo recurso kubectl_manifest
        apiVersion = "metallb.io/v1beta1"
        kind = "IPAddressPool"
        metadata = {
            name = "homelab-ip-pool"
            namespace = "metallb-system"
        }
        spec= {
            addresses = [
                "192.168.0.XXX-192.168.0.XXX"
            ]
        }
    })
}

# cria o recurso L2Advertisement do MetalLB
resource "kubectl_manifest" "metallb_l2_advertisement" {
    depends_on = [kubectl_manifest.metallb_IPAddressPool] # Aguarda o recurso IPAddressPool ser criado antes de criar o recurso L2Advertisement
    yaml_body = yamlencode({
        apiVersion = "metallb.io/v1beta1"
        kind = "L2Advertisement"
        metadata = {
            name = "homelab-l2-advertisement"
            namespace = "metallb-system"
        }
        spec = {
            ipAddressPools = [
                "homelab-ip-pool" # Nome do recurso IPAddressPool criado anteriormente, que será usado pelo L2Advertisement
            ]
        }
    })
}
```
- **2. Aplicar as configuralçoes para o terraform criar os recursos**
```bash
terraform plan # Mosrta o que vai ser alterado

terraform apply # Aplica as alterações
```
>**Nota**: Utilizei o provider `kubectl_manifest` (em vez do
> `kubernetes_manifest` nativo), pois ele adia a validação do schema oara a fase de apply.

---

# Troubleshooting após instalação via Terraforma

### Enfrentei dois problemas clássicos de automação durante a instalação do MetalLB e como os resolvi:

- **1.CRDs no kubernetes_manifest**

**Problema**: Eu estava utilizando o recurso `kubernetes_manifest` do provider oficial. Como a release do Helm e a criação dos recursos do MetalLB estavam no mesmo código, o terraform plan falhava porque esse recurso exige que os `CRDs` (Custom Resource Definitions) já existam no cluster na fase de planejamento.

**Opções de Solução**

🅰️ Separar o código em duas etapas e rodar a criação do `IPAddressPool` e `L2Advertisement` em uma segunda execução.

🅱️ Substituir pelo `kubectl_manifest` `(provider gavinbunney/kubectl)`, pois ele adia a validação do schema para a fase de apply, permitindo passar o YAML dinamicamente via yamlencode.

**Escolha**: Neste momento optei pelo `kubectl_manifest` para manter todo o ciclo de vida da infraestrutura unificado em uma única execução do Terraform.

- **2. Race Condition com o Webhook de validação**

𝗣𝗿𝗼𝗯𝗹𝗲𝗺𝗮: Embora o Helm concluísse a instalação e os Pods estivessem como `Ready`, a criação imediata do IPAddressPool falhava com erros de recusa de conexão TLS.

𝗖𝗮𝘂𝘀𝗮: O controller do MetalLB registra um `ValidatingWebhookConfiguration` na API do Kubernetes que leva alguns segundos após a inicialização dos Pods para começar a aceitar e validar requisições TLS.

𝗦𝗼𝗹𝘂𝗰̧𝗮̃𝗼: Incluí um recurso `time_sleep` `(provider hashicorp/time)` de 30 segundos entre a release do Helm e os manifestos `kubectl_manifest`, encadeado através do bloco `depends_on`.

🚀 𝗥𝗲𝘀𝘂𝗹𝘁𝗮𝗱𝗼: Todo o deploy rodando do zero, 100% automatizado e sem nenhuma intervenção manual!