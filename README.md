# Projeto WordPress com Alta Disponibilidade na AWS

Este projeto implementa uma infraestrutura de **alta disponibilidade**, **segurança** e **escalabilidade automática** para hospedar o WordPress na **AWS**, utilizando **Docker**, **RDS**, **EFS**, **VPC customizada** e **Auto Scaling Group**.

---

## 1. Objetivo

- Garantir **alta disponibilidade** para WordPress  
- Separar **camadas de aplicação, dados e armazenamento**  
- Usar **Docker Compose** para simplificar a execução do WordPress  
- Prover **segurança com IAM, Secrets Manager e Security Groups**  
- Escalar automaticamente com **Auto Scaling Group**

---

## 2. Arquitetura da Solução

### 2.1. Componentes principais

- **Camada de Aplicação (EC2 + Docker Compose)**  
  - Instâncias EC2 executam WordPress em containers  
  - Auto Scaling Group garante elasticidade  
- **Balanceamento (ALB)**  
  - Application Load Balancer distribui o tráfego entre instâncias  
- **Camada de Dados (RDS)**  
  - Banco MySQL no Amazon RDS em sub-redes privadas  
  - Apenas a camada de aplicação pode acessá-lo  
- **Camada de Armazenamento (EFS)**  
  - Diretório `wp-content` montado no Amazon EFS  
  - Todos os conteúdos do WordPress permanecem consistentes  
- **Segurança e Acesso**  
  - Segmentação em **sub-redes públicas e privadas**  
  - **Security Groups** controlam o tráfego  
  - **IAM Role** para EC2 acessar Secrets Manager  
  - **Bastion Host** para acesso administrativo seguro  

### 2.2. Diagrama da Arquitetura

![Diagrama da Arquitetura](./Diagrama.png)

---

## 3. Implementação e Configuração Ideal

### 3.1. Rede (VPC)

1. Crie uma VPC customizada com o assistente **"VPC e mais"**  
2. Configure **duas zonas de disponibilidade (AZs)**  
3. Crie as sub-redes:  
   - 2 públicas (para ALB e Bastion Host)  
   - 2 privadas (para EC2 e RDS)  
4. Configure os gateways:  
   - **Internet Gateway** para saída das sub-redes públicas  
   - **NAT Gateway** para saída das sub-redes privadas

---

### 3.2. Arquivo `.env`

Crie um arquivo `.env` com as variáveis de ambiente para uso no `docker-compose.yml`:

```ini
WORDPRESS_DB_HOST=[SEU_ENDPOINT_RDS]
WORDPRESS_DB_USER=andre
WORDPRESS_DB_PASSWORD=[SENHA_AWS_SECRETS_MANAGER]
WORDPRESS_DB_NAME=wordpress
```

---

### 3.3. Launch Template e User Data

Crie um **Launch Template** com as seguintes configurações:

- **AMI customizada** (Amazon Linux 2 com Docker instalado)
- **Instância t2.micro**
- **Security Group**: `wordpress-ec2-sg`
- **IAM Role**: `EC2-WordPress-Role` com acesso ao Secrets Manager

Adicione o seguinte script no campo **User Data**:

```bash
#!/bin/bash
# Script de inicialização seguro para instâncias WordPress

# Obtém a Região da AWS a partir dos metadados da instância
REGION=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | grep region | awk -F\" '{print $4}')

# --- Variáveis de Configuração ---
SECRET_NAME="wordpress/database/password"
DB_HOST_ENDPOINT="[SEU_ENDPOINT_DO_RDS]"
DB_USER="andre"
DB_NAME="wordpress"
EFS_ID="[SEU_EFS_ID]"

# --- Busca Segura de Credenciais ---
DB_PASSWORD=$(aws secretsmanager get-secret-value --secret-id $SECRET_NAME --region $REGION --query SecretString --output text | awk -F: '{print $2}' | sed 's/,"//' | sed 's/"}//')

# --- Preparação do Ambiente ---
cd /home/ec2-user

# Monta o sistema de ficheiros EFS
sudo mkdir -p /mnt/efs
sudo mount -t efs -o tls ${EFS_ID}:/ /mnt/efs
echo "${EFS_ID}:/ /mnt/efs efs _netdev,tls 0 0" | sudo tee -a /etc/fstab

# --- Execução da Aplicação ---
export WORDPRESS_DB_HOST=${DB_HOST_ENDPOINT}
export WORDPRESS_DB_USER=${DB_USER}
export WORDPRESS_DB_PASSWORD=${DB_PASSWORD}
export WORDPRESS_DB_NAME=${DB_NAME}
docker compose up -d
```

---

### 3.4. Arquivo `docker-compose.yml`

Crie o arquivo `docker-compose.yml` com o seguinte conteúdo:

```yaml
version: "3.8"
services:
  wordpress:
    image: wordpress:latest
    container_name: wordpress
    restart: always
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: ${WORDPRESS_DB_HOST}
      WORDPRESS_DB_USER: ${WORDPRESS_DB_USER}
      WORDPRESS_DB_PASSWORD: ${WORDPRESS_DB_PASSWORD}
      WORDPRESS_DB_NAME: ${WORDPRESS_DB_NAME}
    volumes:
      - /mnt/efs:/var/www/html/wp-content
```

---

### 3.5. Segurança

- **IAM Roles**: EC2 acessa apenas o necessário (Secrets Manager)
- **Segurança em Camadas**:
  - ALB público → EC2 privado → RDS privado
- **EFS com TLS**: garante criptografia no transporte

---

### 3.6. Escalabilidade

- **Auto Scaling Group** aumenta ou diminui instâncias EC2 conforme demanda
- **ALB** garante distribuição automática do tráfego
- **EFS** mantém conteúdo sincronizado entre instâncias

---

### 3.7. Conclusão

Essa arquitetura garante:

✅ **Alta disponibilidade** com múltiplas AZs e Auto Scaling  
✅ **Segurança em múltiplas camadas** com IAM, SGs e rede segmentada  
✅ **Escalabilidade automática** com ASG e ALB  
✅ **Separação clara de responsabilidades** entre aplicação, dados e armazenamento  
✅ **Persistência de dados** com RDS para o banco e EFS para o conteúdo do WordPress
