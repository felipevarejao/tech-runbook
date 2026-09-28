# ArgoCD
É uma ferramenta de entrega continua (CD) baseada em GitOps para k8s.

- Um controlador do k8s que monitora aplicações em execução
- Automatiza a sincronização entre o estado no Git e o cluster real

---
## Instalação do ArgoCD via Terraform
- **1. Primeiro vamos criar o arquivo argocd.tf para uma instalação simples**
```nginx
# Cria o namespace do ArgoCD
resource "kubernetes_namespace_v1" "argocd" {
  metadata {
    name = "argocd"
  }
}

# Instalação do ArgoCD usando o Helm
resource "helm_release" "argocd" {
  depends_on = [kubernetes_namespace_v1.argocd]
  name       = "argocd"
  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-cd"
  version    = "10.9.2"
  wait       = true
  timeout    = 900 # Aguarda até 15 minutos para a instalação do ArgoCD ser concluída
  namespace  = kubernetes_namespace_v1.argocd.metadata[0].name

# Habilita o modo insecure para o ArgoCD, permitindo o acesso sem HTTPS (não recomendado para produção)
  set = [
    {
      name  = "server.extraArgs[0]"
      value = "--insecure"
    }
  ]
}
```
- **2. Aplicando o recurso**
```bash
terrafrom plan

terraform apply
```
---

## Incluindo no arquivo a criação de um ingress
>**Nota**: Neste caso não estou passando o yaml do ingress hardcode e sim por yaml file

- **1. Criar o arquico yaml do ingress**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  ingressClassName: traefik
  rules:
    - host: argocd.local # pode substituir pelo seu DNS
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
```
- **2. Referenciar dentro do arquivo `argocd.tf` para fazer a criação do recurso**
```nginx
# Cria o recurso Ingress do ArgoCD com yaml file
resource "kubectl_manifest" "argocd_ingress" {
  depends_on = [helm_release.argocd]
  yaml_body = file("${path.module}/k8s/ingress-argocd.yaml")
}
```
---

# Autenticação SSO no ArgoCD com GitHub OAuth via Terraform

- **Pré-requisitos**

**OAuth App no GitHub:**

    Homepage URL: http://<seu-dominio-ou-tailscale-ip>
    
    Authorization callback URL: http://<seu-dominio-ou-tailscale-ip>/api/dex/callback

- **Como criar o OAuth App:**

a. Acesse o GitHub

b. Clique na sua foto de perfil no canto superior direito e selecione Settings.

c. Na barra lateral esquerda, role até o final e clique em Developer settings.

d. Clique em OAuth apps na barra lateral esquerda.

e. Clique no botão New OAuth App (ou Register a new application).

f. Preencha o formulário com as seguintes informações:

    Application name: O nome público do seu aplicativo.

    Homepage URL: O endereço do site ou da página principal do seu app.

    Application description (opcional): Uma breve descrição do que seu app faz.

    Authorization callback URL: O endereço para onde o GitHub vai redirecionar o usuário após a autenticação (por exemplo, http://localhost:3000/callback para ambiente de testes).

g.Clique em Register application.


## Obtendo as credenciais

a. Após o registro, você será direcionado para a página do aplicativo criado.

b. Copie o seu Client ID.Clique em Generate a new client secret para criar e copiar a sua chave secreta (Client Secret). 

c. Guarde esse segredo em um local seguro, pois ele não será exibido novamente

---

## Como implementar 
- **1. Criar o arquivo values.yaml**
```yaml
configs:
  cm:
    url: <URL_DNS>
    admin.enabled: false # Desativa o usuário admin padrão
    dex.config: |
      connectors:
        - type: github
          id: github
          name: GitHub
          config:
            clientID: ${github_client_id}
            clientSecret: ${github_client_secret}
  rbac:
    policy.csv: | # Da acesso de admin ao usuário do github
      g, ${github_admin_user}, role:admin 
```

- **2. Crie o arquivo terraform.tfvars com as suas credenciais do GitHub**
```nginx
github_client_id     = "SEU_CLIENT_ID_DO_GITHUB"
github_client_secret = "SEU_CLIENT_SECRET_DO_GITHUB"
github_admin_user    = "SEU_USUARIO_GITHUB"
```

- **3. Referencia os values dentro do arquivo argocd.tf**
```nginx
  # Define os valores do ArgoCD a partir do arquivo values-argocd.yaml
  values = [
    templatefile("${path.module}/k8s/values-argocd.yaml", {
      github_client_id     = var.github_client_id
      github_client_secret = var.github_client_secret
      github_admin_user    = var.github_admin_user
    })
  ]
```
- **Fazer a aplicação do arquivo argocd.tf para criar os recursos**
```bash
terraform plan

terraform apply

# Reinicia os pods do ArgoCD para coletar as novas secrets

kubectl rollout restart deployment/argocd-server -n argocd
kubectl rollout restart deployment/argocd-dex-server -n argocd
```
---

# Como fica o arquivo `argocd.tf` completo
```nginx
# Cria o namespace do ArgoCD
resource "kubernetes_namespace_v1" "argocd" {
  metadata {
    name = "argocd"
  }
}

# Instalação do ArgoCD usando o Helm
resource "helm_release" "argocd" {
  depends_on = [kubernetes_namespace_v1.argocd]
  name       = "argocd"
  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-cd"
  version    = "10.9.2"
  wait       = true
  timeout    = 900 # Aguarda até 15 minutos para a instalação do ArgoCD ser concluída
  namespace  = kubernetes_namespace_v1.argocd.metadata[0].name

  # Habilita o modo insecure para o ArgoCD, permitindo o acesso sem HTTPS (não recomendado para produção)
  set = [
    {
      name  = "server.extraArgs[0]"
      value = "--insecure"
    }
  ]

  # Define os valores do ArgoCD a partir do arquivo values-argocd.yaml
  values = [
    templatefile("${path.module}/k8s/values-argocd.yaml", {
      github_client_id     = var.github_client_id
      github_client_secret = var.github_client_secret
      github_admin_user    = var.github_admin_user
    })
  ]
}

# Cria o recurso Ingress do ArgoCD com yaml file
resource "kubectl_manifest" "argocd_ingress" {
  depends_on = [helm_release.argocd]
  yaml_body = file("${path.module}/k8s/ingress-argocd.yaml")
}
```

---

# Problemas enfrentado na hora da validação

## Erro 404 ao Acessar a Interface do ArgoCD via Ingress (Traefik + Tailscale)

### Sintoma
Ao tentar acessar o painel do ArgoCD pelo navegador utilizando o DNS do Tailscale, o Traefik retornava uma página de erro `404 page not found`.

---

### Causa Raiz
O erro ocorreu devido a um **descasamento de protocolo e criptografia** entre o Ingress Controller (Traefik) e o pod backend do ArgoCD (`argocd-server`):

1. **TLS Ativo no ArgoCD:** Por padrão, o `argocd-server` exige comunicação TLS (HTTPS) rodando internamente na porta 443 com certificados autoassinados.
2. **Conflito de Porta x Protocolo:** Apontar o Ingress para a porta `80` enquanto o backend esperava HTTPS fazia o Traefik falhar na negociação da conexão.
3. **Conflito de Annotations (`h2c` x `HTTPS`):** As anotações do Traefik forçavam o protocolo `h2c` (*HTTP/2 Cleartext* — HTTP/2 sem criptografia). Ao alterar a porta do Ingress para `443`, a anotação `h2c` entrou em conflito direto com a exigência de TLS do ArgoCD.
4. **Rejeição do Certificado Autoassinado:** Sem o modo `--insecure` ativo no ArgoCD, o Traefik recusava a conexão HTTPS com o backend por não confiar no certificado interno, descartando a rota e gerando o erro `404`.

---

###  Solução Aplicada

Para resolver o problema, desacoplei  a criptografia do backend, deixando o `argocd-server` responder em HTTP simplificando o manifesto do Ingress.

#### 1. Habilitar o modo `--insecure` no ArgoCD via Terraform
No arquivo de configuração do Terraform (`argocd.tf` / `values-argocd.yaml`), adicionei a flag `--insecure` nos argumentos do servidor para que ele aceite conexões HTTP puras na porta 80.

```nginx
# Exemplo de configuração no values/terraform do ArgoCD
server = {
  extraArgs = [
    "--insecure"
  ]
}
```
