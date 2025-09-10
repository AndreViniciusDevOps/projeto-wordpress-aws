# Projeto WordPress com Alta Disponibilidade na AWS

Este projeto implementa uma infraestrutura de alta disponibilidade, segurança e escalabilidade automática para hospedar o WordPress na AWS, utilizando Docker, RDS, EFS, VPC customizada e Auto Scaling Group.

## 1. Objetivo

* **Garantir alta disponibilidade** para o WordPress, resistindo a falhas de instâncias ou zonas de disponibilidade.
* **Separar as camadas** de aplicação, dados e armazenamento para melhor gerenciamento e segurança.
* **Usar Docker e Docker Compose** para simplificar a execução e o gerenciamento do ambiente WordPress.
* **Prover segurança robusta** com IAM, Secrets Manager, VPC com sub-redes privadas e Security Groups.
* **Escalar automaticamente** a camada de aplicação com um Auto Scaling Group para lidar com variações de tráfego.

## 2. Arquitetura da Solução

### 2.1. Componentes Principais

* **Camada de Aplicação (EC2 + Docker Compose):** Instâncias EC2 em sub-redes privadas executam o WordPress em containers Docker. Um Auto Scaling Group gerencia a quantidade de instâncias com base na demanda.
* **Balanceamento de Carga (ALB):** Um Application Load Balancer em sub-redes públicas distribui o tráfego de entrada de forma inteligente entre as instâncias EC2 ativas.
* **Camada de Dados (RDS):** Um banco de dados MySQL gerenciado pelo Amazon RDS, configurado em modo Multi-AZ para alta disponibilidade e localizado em sub-redes privadas. Apenas a camada de aplicação pode acessá-lo.
* **Camada de Armazenamento (EFS):** O diretório `wp-content` (que contém plugins, temas e uploads) é montado em um Amazon EFS. Isso garante que todos os containers do WordPress acessem os mesmos arquivos, mantendo a consistência.
* **Segurança e Acesso:**
    * **Rede:** Segmentação em sub-redes públicas (ALB, Bastion Host) e privadas (EC2, RDS).
    * **Firewall:** Security Groups controlam estritamente o tráfego entre as camadas.
    * **Credenciais:** IAM Role para que as instâncias EC2 possam acessar senhas no Secrets Manager de forma segura, sem expor chaves de acesso.
    * **Acesso Administrativo:** Um Bastion Host (ou Systems Manager Session Manager) em uma sub-rede pública permite acesso seguro para gerenciamento.

### 2.2. Diagrama da Arquitetura

![Diagrama da Arquitetura](Diagrama.png)

## 3. Implementação e Configuração Ideal

### 3.1. Rede (VPC)

1.  No Console da AWS, vá para o serviço VPC.
2.  Use o assistente **"VPC e mais"** para uma criação simplificada.
3.  **Configurações:**
    * **Nome:** `wordpress-vpc`
    * **Bloco CIDR IPv4:** `10.0.0.0/16`
    * **Zonas de Disponibilidade (AZs):** Escolha 2 AZs para garantir alta disponibilidade.
    * **Número de sub-redes públicas:** 2 (uma para cada AZ).
    * **Número de sub-redes privadas:** 2 (uma para cada AZ).
    * **NAT Gateways:** Selecione **"1 por AZ"** para permitir que as instâncias nas sub-redes privadas acessem a internet para atualizações (ex: baixar a imagem do Docker).
    * **Endpoints da VPC:** Nenhum.
4.  Clique em **"Criar VPC"**. O assistente criará automaticamente a VPC, sub-redes, tabelas de rotas, Internet Gateway e NAT Gateways.

### 3.2. Camada de Dados (Amazon RDS)

O RDS em modo Multi-AZ cria uma réplica síncrona do seu banco de dados em outra Zona de Disponibilidade, garantindo um failover automático em caso de falha.

**Como configurar:**

1.  No console do RDS, clique em **"Criar banco de dados"**.
2.  **Método de criação:** "Criação padrão".
3.  **Mecanismo:** "MySQL".
4.  **Modelo:** Selecione **"Produção"** para habilitar as opções de alta disponibilidade.
5.  **Configurações:**
    * **Identificador da instância:** `wordpress-db`.
    * **Credenciais:** Defina um usuário (`andre`) e uma senha. **Armazene esta senha imediatamente no AWS Secrets Manager** com o nome `wordpress/database/password`.
6.  **Disponibilidade e durabilidade:** Selecione **"Criar uma instância de banco de dados standby (recomendado para produção)"**. Isso ativa o modo Multi-AZ.
7.  **Conectividade:**
    * **VPC:** Selecione a `wordpress-vpc` criada anteriormente.
    * **Grupo de sub-redes:** O RDS deve criar um novo grupo de sub-redes usando as **sub-redes privadas**.
    * **Acesso público:** **"Não"**.
    * **Security Group:** Crie um novo com o nome `wordpress-rds-sg`. Mais tarde, editaremos sua regra de entrada.
8.  Clique em **"Criar banco de dados"**.
9.  Após a criação, anote o **Endpoint** do banco de dados. Ele será usado no script `User Data`.

### 3.3. Camada de Armazenamento (Amazon EFS)

O EFS fornece um sistema de arquivos de rede compartilhado e escalável, essencial para que todas as instâncias do WordPress vejam os mesmos arquivos de `wp-content`.

**Como configurar:**

1.  No console do EFS, clique em **"Criar sistema de arquivos"**.
2.  **Configuração:**
    * **Nome:** `wordpress-efs`.
    * **VPC:** Selecione a `wordpress-vpc`.
3.  **Acesso à rede (Mount Targets):**
    * Para cada Zona de Disponibilidade que contém uma **sub-rede privada**, adicione um "mount target".
    * **Security Group:** Crie um novo Security Group chamado `wordpress-efs-sg` para os mount targets.
4.  Revise e clique em **"Criar"**.
5.  Após a criação, anote o **ID do sistema de arquivos (File system ID)**. Ele será usado no script `User Data`.

### 3.4. Segurança (Security Groups)

Configure as regras de firewall para controlar o tráfego:

* **`wordpress-alb-sg` (Anexado ao Load Balancer):**
    * **Entrada:** Permite tráfego HTTP (porta 80) e HTTPS (porta 443) de qualquer lugar (`0.0.0.0/0`).
* **`wordpress-ec2-sg` (Anexado às instâncias EC2):**
    * **Entrada:**
        * Permite tráfego na porta 80 vindo **apenas** do Security Group do ALB (`wordpress-alb-sg`).
        * Permite tráfego SSH (porta 22) vindo do seu IP ou do Bastion Host para administração.
* **`wordpress-rds-sg` (Anexado à instância RDS):**
    * **Entrada:** Permite tráfego na porta 3306 (MySQL) vindo **apenas** do Security Group das instâncias EC2 (`wordpress-ec2-sg`).
* **`wordpress-efs-sg` (Anexado aos Mount Targets do EFS):**
    * **Entrada:** Permite tráfego na porta 2049 (NFS) vindo **apenas** do Security Group das instâncias EC2 (`wordpress-ec2-sg`).

### 3.5. Launch Template e User Data

O Launch Template é o "molde" para as instâncias EC2 que o Auto Scaling Group irá criar.

1.  **Crie a IAM Role:** Vá ao IAM e crie uma role chamada `EC2-WordPress-Role`. Anexe a política gerenciada pela AWS `SecretsManagerReadWrite` (ou uma política mais restrita que permita apenas a leitura do segredo específico).
2.  **Crie o Launch Template:**
    * No console do EC2, vá para **"Launch Templates"** e clique em **"Criar launch template"**.
    * **Nome:** `wordpress-launch-template`.
    * **AMI:** Use uma Amazon Linux 2 (ou uma AMI customizada com Docker e Docker Compose já instalados).
    * **Tipo de instância:** `t2.micro` ou superior.
    * **Par de chaves (key pair):** Associe um par de chaves para acesso de emergência.
    * **Configurações de rede:** Não especifique a sub-rede aqui (o ASG cuidará disso). Associe o Security Group `wordpress-ec2-sg`.
    * **Perfil da instância IAM:** Selecione a role `EC2-WordPress-Role`.
    * **Detalhes avançados > User Data:** Adicione o script abaixo, substituindo os placeholders.

**Script User Data:**
Este script é executado toda vez que uma nova instância é iniciada.

```bash
#!/bin/bash
# Script de inicialização seguro para instâncias WordPress

# Instalação de dependências
sudo yum update -y
sudo yum install -y docker git
sudo systemctl enable docker
sudo systemctl start docker

# Instalação do Docker Compose
sudo curl -L "[https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname](https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname) -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Adiciona ec2-user ao grupo docker para executar comandos docker sem sudo
sudo usermod -a -G docker ec2-user

# Obtém a Região da AWS a partir dos metadados da instância
REGION=$(curl -s [http://169.254.169.254/latest/dynamic/instance-identity/document](http://169.254.169.254/latest/dynamic/instance-identity/document) | grep region | awk -F\" '{print $4}')

# --- Variáveis de Configuração (SUBSTITUA OS VALORES) ---
SECRET_NAME="wordpress/database/password"
DB_HOST_ENDPOINT="[SEU_ENDPOINT_DO_RDS]"
DB_USER="andre"
DB_NAME="wordpress"
EFS_ID="[SEU_EFS_ID]"
# --- Fim das Variáveis ---

# --- Busca Segura de Credenciais ---
DB_PASSWORD=$(aws secretsmanager get-secret-value --secret-id $SECRET_NAME --region $REGION --query SecretString --output text | awk -F'":"' '{print $2}' | sed 's/"}//')

# --- Preparação do Ambiente ---
cd /home/ec2-user
git clone [https://github.com/seu-usuario/seu-repositorio-com-docker-compose.git](https://github.com/seu-usuario/seu-repositorio-com-docker-compose.git) app
cd app # Navega para o diretório que contém o docker-compose.yml

# Monta o sistema de ficheiros EFS
sudo mkdir -p /mnt/efs
# A flag -o tls garante criptografia em trânsito
sudo mount -t efs -o tls ${EFS_ID}:/ /mnt/efs
# Garante que o EFS será montado após reinicializações
echo "${EFS_ID}:/ /mnt/efs efs _netdev,tls 0 0" | sudo tee -a /etc/fstab

# --- Execução da Aplicação ---
# Cria o arquivo .env para o Docker Compose usar
cat > .env << EOF
WORDPRESS_DB_HOST=${DB_HOST_ENDPOINT}
WORDPRESS_DB_USER=${DB_USER}
WORDPRESS_DB_PASSWORD=${DB_PASSWORD}
WORDPRESS_DB_NAME=${DB_NAME}
EOF

# Inicia o container em segundo plano
docker compose up -d
```

### 3.6. Balanceamento de Carga (ALB) e Auto Scaling

#### 3.6.1. Configuração do Target Group

1.  No console do EC2, vá para **"Target Groups"** e clique em **"Criar target group"**.
2.  **Tipo de target:** "Instâncias".
3.  **Nome:** `wordpress-tg`.
4.  **Protocolo e Porta:** `HTTP` e `80`.
5.  **VPC:** Selecione a `wordpress-vpc`.
6.  **Health checks (Verificações de saúde):** Deixe o caminho como `/`. O ALB usará isso para verificar se uma instância está saudável antes de enviar tráfego para ela.
7.  Clique em **"Avançar"**. Não registre nenhum target manualmente, o Auto Scaling Group fará isso.

#### 3.6.2. Configuração do Application Load Balancer (ALB)

1.  No console do EC2, vá para **"Load Balancers"** e clique em **"Criar Load Balancer"**.
2.  **Tipo:** "Application Load Balancer".
3.  **Nome:** `wordpress-alb`.
4.  **Scheme:** "Internet-facing" (de frente para a internet).
5.  **VPC:** Selecione a `wordpress-vpc`.
6.  **Mapeamentos:** Selecione as **duas sub-redes públicas**.
7.  **Security Group:** Associe o `wordpress-alb-sg`.
8.  **Listeners e roteamento:**
    * Crie um listener na porta `80` (HTTP).
    * Para a ação padrão, selecione **"Encaminhar para"** e escolha o target group `wordpress-tg` criado anteriormente.
    * (Opcional, mas recomendado): Adicione um listener na porta `443` (HTTPS) usando um certificado do AWS Certificate Manager (ACM) para tráfego criptografado.
9.  Clique em **"Criar load balancer"**.

#### 3.6.3. Configuração do Auto Scaling Group (ASG)

1.  No console do EC2, vá para **"Auto Scaling Groups"** e clique em **"Criar Auto Scaling group"**.
2.  **Nome:** `wordpress-asg`.
3.  **Launch Template:** Selecione o `wordpress-launch-template`.
4.  **Rede:**
    * **VPC:** `wordpress-vpc`.
    * **Sub-redes:** Selecione as **duas sub-redes privadas**. O ASG irá balancear as instâncias entre elas.
5.  **Balanceamento de carga:**
    * Selecione **"Anexar a um load balancer existente"**.
    * Escolha o target group `wordpress-tg`.
6.  **Tamanho do grupo e políticas de escalabilidade:**
    * **Capacidade desejada:** `2` (para iniciar com alta disponibilidade).
    * **Capacidade mínima:** `2`.
    * **Capacidade máxima:** `4` (ou mais, dependendo da sua necessidade).
    * **Política de escalabilidade:**
        * Selecione **"Política de escalabilidade de rastreamento de destino"**.
        * **Métrica:** "Utilização média da CPU".
        * **Valor de destino:** `50`. Isso significa que o ASG adicionará instâncias se a CPU média ficar acima de 50% e removerá instâncias se ficar abaixo.
7.  Siga os passos restantes e clique em **"Criar Auto Scaling group"**.

O ASG agora irá automaticamente lançar 2 instâncias, registrá-las no ALB e monitorar a carga para escalar conforme necessário.

### 3.7. Arquivo `docker-compose.yml`

Este arquivo deve estar no seu repositório Git que o script User Data clona. Ele define o serviço do WordPress.

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
      # Mapeia o diretório compartilhado do EFS para o wp-content do container
      - /mnt/efs:/var/www/html/wp-content
```

## 4. Conclusão

Essa arquitetura garante:

* ✅ **Alta disponibilidade** com múltiplas AZs para todos os componentes críticos (ALB, EC2, RDS) e Auto Scaling para auto-recuperação de instâncias.
* ✅ **Segurança em múltiplas camadas** com IAM, Secrets Manager, Security Groups e uma rede devidamente segmentada.
* ✅ **Escalabilidade automática** para lidar com picos de tráfego sem intervenção manual.
* ✅ **Separação clara de responsabilidades** entre aplicação (EC2), dados (RDS) e armazenamento de arquivos (EFS).
* ✅ **Persistência e consistência de dados** com RDS para o banco e EFS para o conteúdo do WordPress, garantindo que qualquer instância tenha acesso aos mesmos dados.
