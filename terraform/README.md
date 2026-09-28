# Terraform
Uma ferramenta de infraestrutura como código (IaC) que permite criar, alterar e versionar recursos de nuvem de forma segura e prevísivel.

Em vez de executar comandos manuais (`kubectl apply`, `helm install`), a infraestrutura é descrita em arquivos de código (`.tf`).

---
# 1. Instalação no Linux (Ubuntu/Debian)
```bash
# 1. Instalar dependências necessárias
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl

# 2. Adicionar a chave GPG oficial da HashiCorp
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# 3. Adicionar o repositório oficial
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# 4. Atualizar e instalar o CLI do Terraform
sudo apt-get update && sudo apt-get install terraform

# 5. Para validar a instalação
terraform -version
```

# 2. Os principais arquivos de um projeto Terraform
>Dividimos as responsabilidades em arquivos padrão `.tf`:

- `providers.tf`: Define quais plugins/tecnologias o Terraform vai gerenciar (ex: Kubernetes, Helm, AWS) e como se autenticar neles.

- `main.tf` ou `<recurso>.tf` (ex: argocd.tf, longhorn.tf): Onde declaramos os recursos reais que queremos criar (charts Helm, pods, instâncias, etc.).

- `variables.tf`: O "contrato" das variáveis. Define os nomes, tipos (string, map, list) e descrições das variáveis que o projeto aceita.

- `terraform.tfvars`: Contém os valores reais para as variáveis (incluindo credenciais sensíveis). NUNCA deve ir para o Git.

- `outputs.tf` (opcional): Exibe informações úteis no terminal após o apply (ex: URLs criadas, IPs alocados).

- `.terraform.lock.hcl`: Arquivo gerado automaticamente no terraform init para travar as versões exatas dos providers instalados.

## Por que usar Infraestrutura como Código?

* **Reprodutibilidade:** Permite recriar todo o ambiente do zero com apenas um comando (`terraform apply`).
* **Rastreabilidade e GitOps:** Toda alteração de infraestrutura passa por histórico.
* **Gestão de Segredos:** Permite injetar variáveis sensíveis em tempo de execução sem salvá-las em texto puro no repositório.

---

## Conceito Chave: `file()` vs `templatefile()`
Um dos aprendizados mais importantes na integração do Terraform com Kubernetes/Helm:

`file()`: Lê o conteúdo de um arquivo exatamente como ele é (texto estático). Não aceita substituição de variáveis.

`templatefile()`: Lê um arquivo de template (ex: `.yaml`) e substitui as variáveis
`${...}` pelos valores definidos no mapa do Terraform.

## Fluxo de trabalho e comando do dia a dia
```bash
# 1. Inicializar os providers e módulos declarados
terraform init

# 2. Validar a sintaxe do código
terraform validate

# 3. Simular as alterações (Dry-Run) sem alterar o ambiente
terraform plan

# 4. Aplicar as alterações no cluster/ambiente
terraform apply -auto-approve

# 5. Destruir os recursos gerenciados (Cuidado!)
terraform destroy
```

## 3. Boas práticas de segurança

- a. Nunca commitar `.tfvars` com credenciais: Adicione sempre `*.tfvars` ao `.gitignore`.
- b. Salvar arquivos antes do `plan`: Modificações no `terraform.tfvars` só surtem efeito se o arquivo estiver salvo em disco.