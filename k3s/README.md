# Cluster k3s
É uma distribuição do k8s leve e otimizada para baixo consumo de recursos.

## Escolha
é uma boa escolha para dispositivos com pouco recurso (homelab), pois precisa apenas 512MB de RAM e 1 núcleo de CPU.
Já vem com as ferramentas essenciais integradas de fábrica: containerd, CoreDNS, Flannel (rede), Traefik (Ingress) e um balanceador de cargas simples (ServiceLB).

No meu caso, com dois notebooks de 4GB RAM, esse leveza do k3s foi decisica.

> **Nota**: por mais que o k3s ja venha com o LB simples (ServiceLB/klipper-lb), no meu
> caso ele foi substituido pelo MetalLB, mais flexivel para rede bare-metal.


## How-To - Instação com um servidor de control-plane e outro como worker
**1. k3s server, no control-plane**:
```bash
curl -sfL https://get.k3s.io | sh -
```

**2. Pegar o token do server** (usado pra entrar o worker no cluster):
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

**3. No servidor que vai assumir o papel de worker**:
```bash
curl -sfL https://get.k3s.io | K3S_URL=<endereço_do_control-plane> K3S_TOKEN=<token> sh -
```
**4. Validando a instalação**:
>**Rodar o comando abaixo no control-plane, deve retornar os dois nodes (control-plane e worker)**
```bash
kubectl get nodes
# o comando deve retornoar os dois nodes (control-plane e worker) 'Ready'.
NAME                  STATUS   ROLES           AGE     VERSION
k3s-control-plane-1   Ready    control-plane   3d20h   v1.36.4+k3s1
k3s-worker-01         Ready    <none>          3d20h   v1.36.4+k3s1
```


