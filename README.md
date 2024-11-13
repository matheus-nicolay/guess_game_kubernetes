# Projeto Guess Game - Deploy no Amazon EKS com Kubernetes e Nginx Ingress Controller

Este guia fornece instruções para implantar o projeto Guess Game no Amazon EKS usando Kubernetes e configurações específicas como o StatefulSet para PostgreSQL e nginx ingress controller para balanceamento de carga.

## Pré-requisitos: 
- Amazon EKS configurado e rodando.
- Credenciais da AWS configuradas no ambiente para acesso ao cluster EKS.
- Docker Hub para armazenar as imagens Docker.
- Configuração do Ambiente AWS e Ferramentas
- Instalar o kubectl

### Instalar kubectl

O kubectl é a ferramenta de linha de comando para interagir com Kubernetes.

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### Instalar o Helm
O Helm é uma ferramenta de gerenciamento de pacotes para Kubernetes.

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### Configurar as Credenciais da AWS

Configure o aws-cli para conectar-se ao EKS e configurar as credenciais:

```bash
aws configure
```

#### Configurar EKSCTL

EKSCTL é utilizado para facilitar a criação/autorização para utilizar volumes EBS como PV.
Download eksctl:

```bash
# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo mv /tmp/eksctl /usr/local/bin
```


Criação da service account e conexão dela com policy e role da AWS

```bash
eksctl utils associate-iam-oidc-provider --region us-west-2 --cluster guess_game --approve

curl -o ebs-csi-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/docs/example-iam-policy.json
aws iam create-policy --policy-name AmazonEKS_EBS_CSI_Driver_Policy --policy-document file://ebs-csi-policy.json

kubectl apply -f "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.30"

eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster guessgame \ #nome do cluster EKS
  --attach-policy-arn arn:aws:iam::<ID_CONTA>:policy/AmazonEKS_EBS_CSI_Driver_Policy \
  --approve \
  --region us-west-2 #região

helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set serviceAccount.controller.create=false \
  --set serviceAccount.controller.name=ebs-csi-controller-sa

```

Em seguida, obtenha as credenciais do EKS para o kubectl:

```bash
aws eks --region <YOUR_REGION> update-kubeconfig --name <YOUR_EKS_CLUSTER_NAME>
```

### Publicação das Imagens Docker no Docker Hub
Para o Kubernetes usar suas imagens Docker, você precisa construir e enviar as imagens Docker do frontend e backend para o Docker Hub.

Login no Docker Hub (substitua YOUR_USERNAME com seu nome de usuário):

```bash
docker login -u YOUR_USERNAME
```

Gerar e enviar a imagem do backend:

```bash
cd guess_game_kubernetes/src/
docker build -t YOUR_USERNAME/guess-game-backend:latest .
docker push YOUR_USERNAME/guess-game-backend:latest
```

Gerar e enviar a imagem do frontend:

```bash
cd frontend
docker build -t YOUR_USERNAME/guess-game-frontend:latest .
docker push YOUR_USERNAME/guess-game-frontend:latest
```

Nota: Certifique-se de substituir YOUR_USERNAME pelo seu nome de usuário do Docker Hub.

### Instalação do Nginx Ingress Controller com Helm
Usaremos o Helm para instalar o nginx ingress controller, que permitirá o roteamento de tráfego para o frontend e o backend.

Adicionar o repositório do Helm para o nginx ingress:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Instalar o nginx ingress controller como um LoadBalancer:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx --set controller.service.type=LoadBalancer
```

Verifique se o Ingress Controller está rodando:

```bash
kubectl get pods -n ingress-nginx
kubectl get services -n ingress-nginx
```

Nota: Quando estiver rodando em um ambiente como EKS, o serviço de LoadBalancer irá provisionar um balanceador de carga na AWS.

### Aplicação dos Manifestos Kubernetes
Agora que o ingress controller está configurado, aplique os arquivos de manifesto do Kubernetes para o backend, frontend, banco de dados PostgreSQL (usando StatefulSet) e o recurso de Ingress.

```bash
cd ../../kubernetes
```

Aplicar o StatefulSet do PostgreSQL:

```bash
kubectl apply -f db-secrets.yaml
kubectl apply -f database.yaml
```

Aplicar o Deployment, Serviço e HPA do Backend:

```bash
kubectl apply -f backend.yaml
kubectl apply -f backend-hpa.yaml
```

Aplicar o Deployment e Serviço do Frontend:

```bash
kubectl apply -f frontend.yaml
```

Aplicar o Ingress:

```bash
kubectl apply -f ingress.yaml
```

Aplicar o storage.yaml, contendo StorageClass e VPC:

```bash
kubectl apply -f storage.yaml
```

Dica: Após aplicar o Ingress, verifique o IP externo atribuído ao ingress controller para acessar o frontend:

```bash
kubectl get ingress
```

Verificar os Recursos no Cluster:

Use os comandos abaixo para verificar se os recursos foram implantados com sucesso:

```bash
kubectl get pods
kubectl get svc
kubectl get ingress
```