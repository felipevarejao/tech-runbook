# Banco de Dados — PostgreSQL com CloudNativePG (CNPG)

> - **Abordagem escolhida:** Em vez de um Helm chart tradicional/monolítico (ex: Bitnami), escolhi o **CloudNativePG (CNPG)**.

- **Motivo da decisão:** O CNPG utiliza o padrão Operator e Custom Resource Definitions (CRDs) nativas do Kubernetes (`postgresql.cnpg.io/v1`). Ele gerencia o ciclo de vida do PostgreSQL de forma declarativa, oferecendo reconciliação contínua, failover automático e backups nativos sem depender de scripts externos.
---

### Decisões importantes
- **Instâncias (`spec.instances`):** `1`
  - *Decisão técnica de hardware:* Minha infra tem apenas 2 nós físicos, criar uma instancia HA clássica com 3 instâncias seria provisionado obrigatóriamente 2 réplicas na mesma máquina física (já que não há um 3º nó isolado para quorum). Se essa máquina caísse, o quorum seria perdido de qualquer forma.

  - Para economizar RAM/CPU nos notebooks (4GB de RAM cada) e evitar uma ilusão de HA, defini `instances: 1`, deixando a persistência e resiliência comandado pelos volumes replicados do **Longhorn**.

- **Armazenamento:** `storageClass: longhorn` declarado explicitamente na especificação do cluster (`size: 2Gi`).
---


## Implementação no Terraform (`infra/postgres.tf`)

```nginx
# Faz a instalação do operador do postgresql no cluster Kubernetes usando o Helm
resource "helm_release" "cloudnative-pg" {
  name       = "postgresql-operator"
  repository = "https://cloudnative-pg.io/charts"
  chart      = "cloudnative-pg"
  version    = "0.29.1"
  namespace  = "database"
  create_namespace = true
  timeout     = 900
  wait       = true
}

# Criar o Kind: Cluster e o primeiro banco de dados PostgreSQL usando o operador
resource "kubectl_manifest" "postgresql_cluster" {
  depends_on = [helm_release.cloudnative-pg]
  yaml_body = file("${path.module}/k8s/kind-cluster-postgres.yaml")
}
```
---


## Yaml de criação do cluster e initdb
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-cluster
  namespace: database
spec:
  instances: 1
  storage:
    size: 2Gi
    storageClass: longhorn # Estou alocando o storage com o Longhorn
  bootstrap:
    initdb:
      database: <Nome do db>
      owner: <Usuário do db>
```