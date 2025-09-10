# Projeto: WordPress de Alta Disponibilidade na AWS

[cite_start]Este projeto tem como objetivo implantar a plataforma WordPress na nuvem AWS de forma escalável e tolerante a falhas, utilizando os principais serviços gerenciados da AWS para garantir desempenho e disponibilidade[cite: 6]. [cite_start]A arquitetura simula um ambiente de produção real, onde a aplicação é resiliente a interrupções[cite: 27].

## Arquitetura Proposta

A arquitetura final distribuirá a aplicação em múltiplas instâncias EC2 gerenciadas por um Auto Scaling Group (ASG) e um Application Load Balancer (ALB). [cite_start]O Amazon EFS será usado para o armazenamento compartilhado de arquivos, e o Amazon RDS (Multi-AZ) para o banco de dados[cite: 25, 26].

![Diagrama da Arquitetura](https://i.imgur.com/URL_DA_IMAGEM_AQUI.png) 
*(Nota: Para usar a imagem, você pode tirar um print do diagrama no PDF, subir em um site como o [Imgur](https://imgur.com/upload) e colar o link aqui).*

---

## Etapas do Projeto

### 1. Ambiente de Desenvolvimento Local com Docker

[cite_start]A primeira etapa consiste em executar uma instância do WordPress localmente para entender seus componentes e funcionamento[cite: 53].

#### Pré-requisitos
* [Docker Desktop](https://www.docker.com/products/docker-desktop/) instalado e em execução.

#### Configuração

O ambiente local utiliza Docker Compose para orquestrar dois serviços: um para o banco de dados `mysql:5.7` e outro para a aplicação `wordpress:latest`.

Para proteger informações sensíveis, as senhas não são escritas diretamente no arquivo `docker-compose.yml`. Em vez disso, elas são gerenciadas através de um arquivo `.env`.

#### Como Executar

1.  **Configure as variáveis de ambiente:**
    Crie um arquivo chamado `.env` na raiz do projeto. Este arquivo **não deve** ser versionado no Git. Adicione a seguinte variável a ele:
    ```
    MYSQL_PASSWORD=SUA_SENHA_FORTE_AQUI
    ```

2.  **Inicie os contêineres:**
    No terminal, navegue até a pasta do projeto e execute o comando:
    ```bash
    docker-compose up -d
    ```

3.  **Acesse a aplicação:**
    Após a inicialização, o WordPress estará acessível no seu navegador em [http://localhost:8080](http://localhost:8080).

### 2. Criação da Infraestrutura na AWS (Em andamento)

* [ ] **Criar a VPC:**
    * [cite_start][ ] 2 Zonas de Disponibilidade[cite: 57].
    * [cite_start][ ] 4 subnets (2 públicas, 2 privadas)[cite: 59].
    * [cite_start][ ] Internet Gateway (IGW)[cite: 60].
    * [cite_start][ ] NAT Gateway nas subnets públicas[cite: 61].
* [cite_start][ ] **Criar o Banco de Dados (RDS)**[cite: 62].
* [cite_start][ ] **Criar o Sistema de Arquivos (EFS)**[cite: 66].
* [cite_start][ ] **Criar o Launch Template e o Auto Scaling Group (ASG)**[cite: 73, 75].
* [cite_start][ ] **Criar o Application Load Balancer (ALB)**[cite: 82].

---