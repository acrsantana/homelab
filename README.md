# homelab
Segue abaixo procedimento detalhado para a montagem do ambiente do homelab Cezão da Bahia. Todo o ambiente foi montado considerando o Ubuntu Server 24.04 como sistema operacional de base, portanto os comandos são baseados neste ambiente.
## Configurar o firewall do servidor
```Bash
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```
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
### Criar um API Token no Cloudflare
No painel do Cloudflare, vá em "Meus Perfil" -> "Tokens de API" -> "Criar Token".  
Use o modelo (template) "Editar DNS de zona".  
Em "Permissões de Zona", selecione "Zona" -> "DNS" -> "Editar".  
Em "Recursos de Zona", selecione "Incluir" -> "Zona Específica" -> e escolha seu meudominio.com.  
Continue, crie o token e copie o valor do token secreto. Você só o verá uma vez.  
***API Token armazenada no 1password***
### Crie o manifest para a secret do cloudflare api token cloudflare-secret.yaml:
```Yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager # Importante: tem que ser no mesmo namespace do cert-manager
type: Opaque
stringData:
  api-token: "COLE_SEU_TOKEN_SECRETO_AQUI" # <-- MUDE AQUI
```
Aplique o secret criado:
```Bash
sudo k3s kubectl apply -f cloudflare-secret.yaml
```
### Crie o Emissor de Certificados (ClusterIssuer):
Crie um arquivo cluster-issuer-dns.yaml:
```Yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod-dns
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: "seu-email-aqui@gmail.com" # <-- MUDE AQUI
    privateKeySecretRef:
      name: letsencrypt-prod-dns-key
    solvers:
    - dns01:
        cloudflare:
          email: "seu-email-aqui@gmail.com" # O e-mail da sua conta Cloudflare
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
```
Aplique o arquivo cluster-issuer-dns.yaml
```Bash
sudo k3s kubectl apply -f cluster-issuer-dns.yaml
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
Reinicialize o servidor:
```Bash
sudo shutdown -r now
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
## Instalar o Docker Registry
### Criar o Diretório de Dados no Servidor
Crie a pasta abaixo no servidor:
```Bash
sudo mkdir -p /data/registry
```
### Crie o Segredo de Autenticação (Login e Senha)
Vamos criar um Secret no K8s para guardar o login e senha que o Traefik (Ingress) usará para proteger seu registry.  
Primeiro, você precisa gerar um htpasswd. O Traefik espera esse formato.  
Vá para um gerador de htpasswd online.  
Digite um usuário (ex: admin) e uma senha segura.  
Ele vai gerar uma linha parecida com: admin:$apr1$b55/s..$EXAMPLE...  
Copie essa linha gerada.  
Agora, crie um arquivo chamado registry-auth.yaml, substituindo a chave stringData.users pela informação gerada pelo gerador de htpasswd:
```Yaml
apiVersion: v1
kind: Namespace
metadata:
  name: homelab-tools
  labels:
    name: homelab-tools
---
apiVersion: v1
kind: Secret
metadata:
  name: registry-auth
  namespace: homelab-tools
stringData:
  users: |
    admin:$apr1$m9s8loz0$LJfWUMT9TwCRKOUFgSAq0/
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: registry-volume
  namespace: homelab-tools
  labels:
    type: local
    project: fractal
    app: registry
spec:
  storageClassName: local-path
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/registry
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-volume-claim
  namespace: homelab-tools
  labels:
    app: registry
    project: fractal
spec:
  storageClassName: local-path
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: docker-registry
  namespace: homelab-tools
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
        project: fractal
    spec:
      containers:
        - name: registry
          image: registry:3.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5000
          volumeMounts:
            - mountPath: /var/lib/registry
              name: registry-data
      volumes:
        - name: registry-data
          persistentVolumeClaim:
            claimName: registry-volume-claim
---
apiVersion: v1
kind: Service
metadata:
  name: registry
  namespace: homelab-tools
  labels:
    app: registry
spec:
  type: ClusterIP
  ports:
    - name: registry-port
      port: 5000
      protocol: TCP
      targetPort: 5000
  selector:
    app: registry
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: registry-ingress
  namespace: homelab-tools
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod-dns"
    traefik.ingress.kubernetes.io/auth.type: basic
    traefik.ingress.kubernetes.io/auth.secret: registry-auth
    traefik.ingress.kubernetes.io/proxy-body-size: "0" 
spec:
  rules:
  - host: "registry.cezaodabahia.com.br"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: registry
            port:
              number: 5000
  tls:
  - hosts:
    - "registry.cezaodabahia.com.br"
    secretName: registry-tls-secret
```
Para usar o registry de dentro do cluster:
1.  **Crie o `imagePullSecret` NO NAMESPACE que deve utilizar a imagem. Ex:`fractal`:**
O segredo deve existir no namespace que vai *usar* a imagem.
```bash
k3s kubectl create secret docker-registry reg-auth-pull \
  --namespace=fractal \
  --docker-server=registry.cezaodabahia.com.br:8443 \
  --docker-username=admin \
  --docker-password=SUA-SENHA-SEGURA-AQUI
```
A senha está armazenada no 1password  
2.  **Use o Segredo no seu Deployment (no namespace `fractal`):**
Qualquer `Deployment` no namespace `fractal` deve ficar assim:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minha-api
  namespace: fractal
spec:
  template:
    spec:
      imagePullSecrets:
        - name: reg-auth-pull
      containers:
        - name: api
          image: "registry.cezaodabahia.com.br:8443/sua-api-fractal:v1"
