# Deploy Escalável do WordPress na AWS com Classic Load Balancer (CLB)

Este documento fornece um guia detalhado para configurar e implantar o WordPress na AWS, utilizando uma arquitetura robusta com **EC2, RDS, EFS, Classic Load Balancer e Auto Scaling Group**. O objetivo é criar uma instalação do WordPress altamente **disponível, escalável e segura**.

## Visão Geral da Arquitetura

- **VPC/Sub-redes**: Rede isolada com sub-redes **públicas** (para CLB, NAT Gateway) e **privadas** (para EC2, RDS, EFS).
- **NAT Gateway**: Permite que instâncias EC2 em sub-redes privadas acessem a internet sem ter IP público.
- **Security Groups (SG)**: Firewalls virtuais para CLB, EC2, RDS e EFS.
- **EC2 (sem IP Público)**: Servidores virtuais executando WordPress (via Docker) em sub-redes privadas.
- **RDS**: Banco de dados MySQL gerenciado em sub-rede privada.
- **EFS**: Armazenamento compartilhado para arquivos WordPress, acessível pelas instâncias EC2.
- **Auto Scaling Group (ASG)**: Gerencia o número de instâncias EC2 baseado em políticas de escalabilidade.
- **Classic Load Balancer (CLB)**: Distribui o tráfego de entrada para as instâncias registradas no ASG.
- **Docker & Docker Compose**: Para conteinerizar a aplicação WordPress.

---

## Configuração Passo a Passo

### 1. Configurar a Rede (VPC e Componentes)

1. **Criar a VPC:**
   - Nome: `wordpress-vpc`, CIDR: `10.0.0.0/16`.

2. **Criar Sub-redes:**
   - **Públicas:** `Public-Subnet-AZ1` (ex: `10.0.1.0/24`), `Public-Subnet-AZ2` (ex: `10.0.2.0/24`).
   - **Privadas:** `Private-Subnet-AZ1` (ex: `10.0.101.0/24`), `Private-Subnet-AZ2` (ex: `10.0.102.0/24`).

3. **Criar Internet Gateway (IGW):**
   - Nome: `wordpress-igw`.
   - Associar à VPC `wordpress-vpc`.

4. **Criar NAT Gateway:**
   - Nome: `wordpress-nat-gw`.
   - Sub-rede: `Public-Subnet-AZ1`.
   - Alocar um IP Elástico e associar ao NAT Gateway.

5. **Configurar Tabelas de Rotas:**
   - **Tabela Pública:** Associar sub-redes públicas e adicionar rota `0.0.0.0/0` -> `wordpress-igw`.
   - **Tabela Privada:** Associar sub-redes privadas e adicionar rota `0.0.0.0/0` -> `wordpress-nat-gw`.

6. **Criar Security Groups (SGs):**
   - **CLB:** Permite tráfego HTTP/HTTPS de `0.0.0.0/0`.
   - **EC2:** Permite tráfego **somente** do CLB, acesso SSH restrito ao IP do admin.
   - **RDS:** Permite conexão apenas das instâncias EC2.
   - **EFS:** Permite montagem de arquivos via NFS pelas instâncias EC2.

---

### 2. Criar o Sistema de Arquivos Compartilhado (EFS)

1. Criar sistema de arquivos **EFS** com nome `wordpress-efs` na VPC `wordpress-vpc`.
2. Associar o **Security Group (efs-sg)**.
3. Criar **Mount Targets** para as sub-redes privadas.

---

### 3. Criar o Banco de Dados Gerenciado (RDS)

1. Criar banco de dados **MySQL** no **RDS** com:
   - Nome: `wordpress-db`
   - Usuário: `admin`
   - Senha: `SUA_SENHA_SEGURA`
   - VPC: `wordpress-vpc`
   - Sub-redes privadas e Security Group `rds-sg`.
2. Anotar o **Endpoint** do banco de dados.

---

### 4. Criar o Launch Template para EC2

1. Criar um **Launch Template** `wordpress-launch-template` com:
   - AMI: **Amazon Linux 2** ou **AL2023**.
   - Tipo de instância: `t2.micro`.
   - Security Group: `ec2-sg`.
2. Adicionar **User Data**:
   ```bash
   #!/bin/bash
   dnf update -y
   dnf install -y docker nfs-utils
   systemctl start docker
   systemctl enable docker
   
   curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   chmod +x /usr/local/bin/docker-compose
   ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
   
   EFS_ID="fs-xxxxxxxx"
   RDS_HOST="wordpress-db.xxxxx.rds.amazonaws.com"
   RDS_USER="admin"
   RDS_PASSWORD="arthur123"
   RDS_NAME="wordpress"
   
   mkdir -p /mnt/efs
   echo "${EFS_ID}:/ /mnt/efs nfs4 defaults,_netdev 0 0" >> /etc/fstab
   mount -a
   mkdir -p /mnt/efs/wordpress
   chown -R 1000:1000 /mnt/efs/wordpress
   chmod -R 777 /mnt/efs/wordpress
   
   mkdir -p /opt/wordpress
   cd /opt/wordpress
   
   cat > docker-compose.yml <<EOF
   services:
     wordpress:
       image: wordpress:latest
       ports:
         - "80:80"
       restart: always
       environment:
         WORDPRESS_DB_HOST: $RDS_HOST
         WORDPRESS_DB_USER: $RDS_USER
         WORDPRESS_DB_PASSWORD: $RDS_PASSWORD
         WORDPRESS_DB_NAME: $RDS_NAME
       volumes:
         - /mnt/efs/wordpress:/var/www/html/wp-content
   EOF
   
   docker-compose up -d
   ```

---

### 5. Criar o Auto Scaling Group (ASG)

1. Criar um **Auto Scaling Group** `wordpress-asg` com:
   - **Launch Template:** `wordpress-launch-template`.
   - **Sub-redes privadas**.
   - **Tamanho desejado:** 2 instâncias.
   - **Política de escalabilidade:** Auto ajuste com CPU acima de **60%**.

---

### 6. Criar o Classic Load Balancer (CLB)

1. Criar um **Classic Load Balancer** `wordpress-clb` com:
   - **Listeners:** HTTP.
   - **Sub-redes públicas**.
   - **Security Group:** `clb-sg`.
   - **Health Check:** `HTTP /wp-admin/install.php`.
2. **Anexar o CLB ao Auto Scaling Group** `wordpress-asg`.

---

### 7. Teste e Monitoramento

- Acesse o **Nome DNS** do Load Balancer para testar o WordPress.
---
