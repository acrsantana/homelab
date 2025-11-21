# homelab
Segue abaixo procedimento detalhado para a montagem do ambiente do homelab Cezão da Bahia. Todo o ambiente foi montado considerando o Ubuntu Server 24.04 como sistema operacional de base, portanto os comandos são baseados neste ambiente.

## Instalação do K3s
### Atualizar pacotes
```Bash
sudo apt update && sudo apt upgrade -y
```
### Instalar o ohmypoosh
```Bash
sudo apt install zip unzip && curl -s https://ohmyposh.dev/install.sh | bash -s
```
### Criar diretórios para snapshots e backups do etcd
```Bash
sudo mkdir /data && sudo mkdir /data/etcd && sudo mkdir /data/etcd/etcd-snapshots && sudo mkdir /data/etcd/etcd-backups && sudo chmod -R 777 /data
```
### Instalar o K3s configurando o intervalo dos snapshots (12h) e o total de retenção (168)
```Bash
sudo curl -sfL https://get.k3s.io | sh -s server - --cluster-init --write-kubeconfig-mode 644 --data-dir=/data/etcd/etcd-backups --etcd-snapshot-retention=168 --etcd-snapshot-dir=/data/etcd/etcd-snapshots --etcd-snapshot-schedule-cron="*/12 * * * *"
```
### Criar o cert manager
```Bash
sudo k3s kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.19.1/cert-manager.yaml
```
### Modificar o .bashrc para adicionar itens importantes
Edite o arquivo .bashrc
```Bash
cd && nano .bashrc
```
Inserir as seguintes linhas:
```Bash
alias k=kubectl
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
export PATH="$PATH:~/.local/bin"
curl https://raw.githubusercontent.com/JanDeDobbeleer/oh-my-posh/main/themes/jandedobbeleer.omp.json -o jandedobbeleer.omp.json
eval "$(oh-my-posh init bash --config 'jandedobbeleer')"
```
### Instalar o Helm
```Bash
sudo snap install helm --classic
```
### Instalar Prometheus e Grafana
Adicionar o Helm Chart do Prometheus:
```Bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
Clonar o repositório abaixo:
```Bash
cd && git clone https://github.com/cablespaghetti/k3s-monitoring.git && cd k3s-monitoring
```
Editar o arquivo kube-prometheus-stack-values.yaml inserindo informações relevantes. Substituir o kube-state-metrics pelo registry.k8s.io/kube-state-metrics/kube-state-metrics:2.17.0
```Bash
cd && cd k3s-monitoring && nano kube-prometheus-stack-values.yaml
```
Instalar o Helm Chart:
```Bash
helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
  --version 39.13.3 \
  --values kube-prometheus-stack-values.yaml \
  --namespace monitoring \
  --create-namespace
```
Aplique o manifest abaixo para configuraro ingress do Grafana
```Bash
k apply -f https://raw.githubusercontent.com/acrsantana/AmbCQ-GitOps/refs/heads/main/grafana-ingress.yaml
```
Aguardar 1-2 minutos para o cert-manager buscar o certificado SSL.
### Acessar a console do Grafana
Acesse a console do Grafana na seguinte url: https://grafana.cezaodabahia.com.br:8443/  
Usuário: admin  
Senha: prom-operator
