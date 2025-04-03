# **Guia Completo: Deploy Escalável do WordPress na AWS com Classic Load Balancer**

Este guia passo a passo explica como configurar e implantar o WordPress na AWS utilizando uma arquitetura escalável e altamente disponível. O projeto inclui VPC, EC2 (sem IP público), RDS, EFS, Auto Scaling Group, Classic Load Balancer (CLB) e monitoração com CloudWatch.

---
## **1. Configurar a Rede (VPC e Sub-redes)**
Antes de iniciar, criamos uma VPC personalizada para isolar nossa infraestrutura.

### **Criando a VPC e Sub-redes**
1. Acesse o console da AWS e abra o serviço **VPC**.
2. Clique em **Criar VPC e mais**.
3. Defina os seguintes parâmetros:
   - Nome: `wordpress-vpc`
   - CIDR: `10.0.0.0/16`
   - Número de zonas de disponibilidade (AZs): `2`
   - Número de sub-redes públicas: `2`
   - Número de sub-redes privadas: `2`
   - Gateways NAT: `Em 1 AZ`
4. Clique em **Criar VPC**.

### **Configurando Rotas**
- A Tabela de Rotas Pública deve ter:
  - Destino `0.0.0.0/0` apontando para o **Internet Gateway (IGW)**.
- A Tabela de Rotas Privada deve ter:
  - Destino `0.0.0.0/0` apontando para o **NAT Gateway** (permite acesso à internet para baixar pacotes, mas impede conexões externas).

---
## **2. Criar o Sistema de Arquivos Compartilhado (EFS)**

O Amazon EFS será usado para armazenar arquivos compartilhados entre as instâncias do WordPress.

1. No console da AWS, acesse o serviço **EFS**.
2. Clique em **Criar sistema de arquivos**.
3. Configure:
   - Nome: `wordpress-efs`
   - VPC: `wordpress-vpc`
   - Modo de desempenho: **Padrão**
   - Modo de taxa de transferência: **Bursting**
4. Crie **Mount Targets** para todas as sub-redes privadas.
5. Conceda permissões no **Security Group (efs-sg)** para permitir conexões NFS das instâncias EC2.

---
## **3. Criar o Banco de Dados Gerenciado (RDS)**

O RDS armazenará os dados do WordPress.

1. Acesse o serviço **RDS**.
2. Clique em **Criar banco de dados**.
3. Escolha **MySQL** e opte por "Criar configuração padrão".
4. Configure:
   - Nome: `wordpress-db`
   - Usuário: `admin`
   - Senha: **Defina uma senha segura**
   - VPC: `wordpress-vpc`
   - Sub-redes privadas
5. Copie o **Endpoint do RDS**, pois ele será usado no WordPress.

---
## **4. Criar a Instância EC2 para o WordPress**

1. No console da AWS, acesse o serviço **EC2**.
2. Clique em **Launch Templates** e depois em **Create Launch Template**.
3. Configure:
   - Nome: `wordpress-launch-template`
   - AMI: **Amazon Linux 2023**
   - Tipo de instância: `t2.micro`
   - Security Group: `ec2-sg` (permitindo tráfego HTTP do Load Balancer)
   - User Data:
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
   RDS_PASSWORD="SUA_SENHA"
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
## **5. Criar o Auto Scaling Group (ASG)**

1. No console da AWS, acesse **EC2 > Auto Scaling Groups**.
2. Clique em **Create Auto Scaling Group**.
3. Configure:
   - Nome: `wordpress-asg`
   - Launch Template: `wordpress-launch-template`
   - Sub-redes privadas
   - Mínimo: `2` instâncias
   - Máximo: `4` instâncias
   - Política de escalonamento: **Escalar quando CPU > 60%**

---
## **6. Criar o Classic Load Balancer (CLB)**

1. No console da AWS, acesse **EC2 > Load Balancers**.
2. Clique em **Create Load Balancer** e escolha **Classic Load Balancer**.
3. Configure:
   - Nome: `wordpress-clb`
   - Listeners: **HTTP (80)**
   - Sub-redes públicas
   - Security Group: `clb-sg`
   - Health Check: `HTTP /wp-admin/install.php`
   - Adicione o **ASG** para receber tráfego

---
