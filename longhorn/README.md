# Longhorn
Os pods no kubernetes são efêmeros, se um pod morre ou muda de nó, ele perde qualquer dado gravado localmente.
Em cloud, existem os dicos gerenciados (como Azure Disk) que resolvem isso. Em bare-metal, não existe esse recurso nativo.

Então o Longhorn entra para gerenciar e replicar os discos entre os nós do cluster. Assim, caso um pod do Postgres que esteja rodando no worker morrer, ele pode subir no control-plane e continuar acessando os mesmo dados.

## Pré-requisitos para instalar o longhorn
> É preciso que seja instalado no servidor que será control-plane e no worker

```bash
sudo apt install -y open-iscsi nfs-common
sudo systemctl enable --now iscsid
```
## How-To de instalação apenas no node control-plane com `kubectl apply`

- **1. Instalação**
```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.7.2/deploy/longhorn.yaml
```

- **2. Validar a instalação**

    - **Criar um `PersistentVolumeClaim` de teste (`storageClassName: longhorn`, 1Gi), confirmado `STATUS: Bound` (não`Pending`)**
```bash
kubectl create pvc longhorn-teste --storage-class=longhorn --access-mode=ReadWriteOnce --capacity=1Gi

# Após validar a criação, podemos deletar o recurs
kubectl delete pvc longhorn-teste
```
## How-To de instalação apenas no node control-plane com `Terraform`

- **1. Criação do arquivo .tf**
```nginx
resource "helm_release" "longhorn" {
   name       = "longhorn"
   repository = "https://charts.longhorn.io"
   chart      = "longhorn"
   version    = "1.12.1"
   namespace  = "longhorn-system"
   create_namespace = true
   wait       = true
   timeout    = 900
}
```
- **2. Aplicar as configuralçoes para o terraform criar os recursos**
```bash
terraform plan # Mosrta o que vai ser alterado

terraform apply # Aplica as alterações
```
- **3. Validar a instalação**

    - **Criar um `PersistentVolumeClaim` de teste (`storageClassName: longhorn`, 1Gi), confirmado `STATUS: Bound` (não`Pending`)**
```bash
kubectl create pvc longhorn-teste --storage-class=longhorn --access-mode=ReadWriteOnce --capacity=1Gi

# Após validar a criação, podemos deletar o recurso
kubectl delete pvc longhorn-teste
```
---

# Comandos de Operação & Troubleshooting

- **Verificar o status dos pods do Longhorn:**
```bash
kubectl get pods -n longhorn-system
```
- **Erro comum - PVC preso em Pending:**
>Geralmente indica que o serviço `iscsid` não está rodando em um dos nós do cluster.
```bash
# Validar o serviço do iSCSI no nó afetado:
sudo systemctl status iscsid
```