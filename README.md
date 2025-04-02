# Documentação: Deploy do WordPress na AWS

Este documento fornece um guia detalhado para configurar e implantar o WordPress na AWS utilizando EC2, RDS, EFS, Auto Scaling e Load Balancer.

---

## Etapa 1: Criar a VPC e Configurar Redes

1. **Acessar o console da AWS**  
   - Faça login no console da AWS.
   - Vá até o serviço **VPC**.

2. **Criar uma VPC**  
   - No painel de VPC, clique em **Criar VPC**.
   - Escolha um nome descritivo, como `wordpress-vpc`.
   - Defina um bloco CIDR, por exemplo: `10.0.0.0/16`.
   - Clique em **Criar VPC**.

3. **Criar Sub-redes**  
   - Vá até **Sub-redes** > **Criar Sub-rede**.
   - Escolha a VPC criada anteriormente.
   - Crie duas sub-redes públicas e duas privadas:
     - Subnet pública 1
       - Name tag: Public-Subnet-1
       - Availability Zone: Escolha uma (exemplo: us-east-1a)
       - IPv4 CIDR block: 10.0.1.0/24
      
     - Subnet pública 2
       - Name tag: Public-Subnet-2
       - Availability Zone: Escolha uma (exemplo: us-east-1b)
       - IPv4 CIDR block: 10.0.2.0/24

     - Subnet privada 1
       - Name tag: Private-Subnet-1
       - Availability Zone: Escolha uma (exemplo: us-east-1a)
       - IPv4 CIDR block: 10.0.3.0/24
    
     - Subnet pública 2
       - Name tag: Private-Subnet-2
       - Availability Zone: Escolha uma (exemplo: us-east-1b)
       - IPv4 CIDR block: 10.0.4.0/24
      
4. **Criar um Internet Gateway**  
   - Vá até **Internet Gateways** > **Criar Internet Gateway**.
   - Nomeie como `wordpress-igw`.
   - Associe à `wordpress-vpc`.

5. **Criar Tabela de Rotas**  
   - Vá para **Tabelas de Rotas** e clique em **Criar Tabela de Rotas**.
   - Nomeie como `public-route-table`.
   - Associe à `wordpress-vpc`.
   - Adicione uma rota com destino `0.0.0.0/0` apontando para o **Internet Gateway** criado.
   - Associe a tabela de rotas à **sub-rede pública**.
     - Vá para a aba Subnet Associations
     - Clique em Edit subnet associations
     - Selecione Public-Subnet-1 e Public-Subnet-2
     - Salve

6. **Criar  Security Group**
   1 - Security Group para EC2
      - Vá até **Security Groups** > **Criar Security Group**.
      - Nomeie como `wordpress-sg` e associe à VPC criada.
      - Adicione as seguintes regras:
        - **HTTP (80)**: Acesso de qualquer lugar (`0.0.0.0/0`)
        - **HTTPS (443)**: Acesso de qualquer lugar (`0.0.0.0/0`)
        - **SSH (22)**: Acesso de qualquer lugar (`0.0.0.0/0`)
      - Clique em Create Security Group.

   2 - Security Group para o Banco de Dados (RDS)
      - Crie um novo grupo de segurança chamado RDS-SG
      - Associe à VPC
      - Adicione as seguintes regras:
        - MySQL/Aurora (3306) – Apenas do WordPress-SG
      - Clique em Create Security Group.

---

## Etapa 2: Criar o EFS para Armazenamento Compartilhado

1. **Acesse o console do EFS**.
2. Clique em **Criar sistema de arquivos**.
3. Defina um nome (`wordpress-efs`)..
4. Selecione a VPC criada anteriormente
5. Clique em Customize
6. Escolha as sub-redes privadas.
7. No Security Group, selecione `wordpress-sg`.
8. Finalize a criação e copie o **ID do EFS**.

---

## Etapa 3: Criar o Banco de Dados RDS

1. **Acesse o console RDS**.
2. Clique em **Criar banco de dados**.
3. Escolha **MySQL**.
4. No método de criação, escolha **Padrão**.
5. Nomeie o banco de dados (`wordpress-db`).
6. Defina um usuário administrador (`admin`).
7. Defina uma senha segura.
8. Escolha a opção **Somente rede VPC** e selecione a VPC criada.
9. No **Security Group**, selecione `wordpress-sg`.
10. Conclua a criação e anote o **endpoint** do banco de dados.

---

## Etapa 4: Criar a Instância EC2 e Configurar User Data

1. **Acesse o console do EC2**.
2. Clique em **Executar Instância**.
3. Escolha uma imagem **Amazon Linux 2**.
4. Escolha o tipo de instância (`t2.micro` para testes).
5. Associe a instância à **sub-rede pública**.
6. No Security Group, selecione `wordpress-sg`.
7. Em **User Data**, cole o seguinte script:

```bash
#!/bin/bash
dnf update -y
dnf install -y docker nfs-utils
systemctl start docker
systemctl enable docker

curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

EFS_ID="fs-xxxxxxxx.efs.us-east-1.amazonaws.com"
RDS_HOST="wordpress-db.xxxxx.us-east-1.rds.amazonaws.com"
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

## Etapa 5: Criar um Target Group e Load Balancer

1. Vá para EC2 → No menu lateral, clique em Target Groups
2. Clique em Create Target Group
3. Preencha os campos:
   - Choose a target type → Selecione Instances
   - Target group name → wordpress-target-group
   - Protocol → HTTP
   - Port → 80
   - VPC → Selecione a mesma VPC da sua aplicação
   - Health Check Settings
     - Protocol → HTTP
     - Path → /
     - Success Codes → 200
4. Clique em Next
5. Registrar Instâncias:
   - Selecione suas instâncias EC2 rodando o WordPress
   - Clique em Include as pending below
   - Depois, clique em Create target group
6. No Console AWS, vá para EC2 → Load Balancers
7. Clique em Create Load Balancer
8. Escolha Application Load Balancer
9. Configure as opções:
   - Name → wordpress-load-balancer
   - Scheme → Internet-facing (para acesso público)
   - IP address type → IPv4
   - VPC → Selecione sua VPC
   - Availability Zones → Escolha pelo menos duas zonas de disponibilidade
10. Listeners and Routing
   - Protocol → HTTP
   - Port → 80
   - Default action → Forward to wordpress-target-group
11. Security Groups
   - Crie um novo Security Group com as seguintes regras:
     - Type: HTTP
     - Protocol: TCP
     - Port Range: 80
     - Source: 0.0.0.0/0 (Acesso público)
12. Clique em Create Load Balancer
13. Copie o DNS Name do Load Balancer para testar depois
---

## Etapa 6: Configurar Auto Scaling Group

1. **Acesse o console do EC2**.
2. Vá até **Auto Scaling Groups** > **Criar Auto Scaling Group**.
3. Defina um nome (`wordpress-asg`).
4. Launch template → Clique em Create a launch template
5. Launch Template Name → wordpress-launch-template
6. AMI (Amazon Machine Image) → Selecione a mesma imagem da sua instância atual
7. Instance type → Escolha o mesmo tipo da sua EC2
8. Key Pair → Selecione a chave SSH
9. Network settings:
   - VPC → Escolha sua VPC
   - Subnets → Escolha múltiplas Availability Zones
   - Security Group → O mesmo que está na sua instância atual
10. User data
   - Copie o User Data que você usou para iniciar a instância EC2.
11. Clique em Create launch template
12. Voltar ao Auto Scaling Group
13. Choose launch template → Selecione wordpress-launch-template
14. Network → Escolha a VPC correta
15. Load Balancing:
   -Escolha Attach to an existing load balancer
   -Selecione o Target Group criado anteriormente
16. Health Check Type → Selecione ELB
17. Group size:
   - Desired capacity: 2
   - Minimum capacity: 2
   - Maximum capacity: 5
18. Scaling Policies → Escolha Target Tracking Scaling Policy
   - Metric Type → "Average CPU utilization"
   - Target Value → 50%
19. Clique em Create Auto Scaling Group

---

## Etapa 7: Atualizar WP_HOME e WP_SITEURL

1. **Acesse a instância EC2** via SSH.
2. Edite o arquivo `wp-config.php`:
   ```bash
   nano /mnt/efs/wordpress/wp-config.php
   ```
3. Atualize as seguintes linhas:
   ```php
   define('WP_HOME', 'http://SEU-LOAD-BALANCER');
   define('WP_SITEURL', 'http://SEU-LOAD-BALANCER');
   ```
4. Reinicie o container:
   ```bash
   docker-compose down && docker-compose up -d
   ```

---

## Etapa 8: Testar a Instalação

1. Acesse `http://SEU-LOAD-BALANCER`.
2. Conclua a instalação do WordPress.
3. Pronto!
