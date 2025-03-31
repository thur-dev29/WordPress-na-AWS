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
   - Crie pelo menos duas sub-redes:
     - **Sub-rede pública**: `10.0.1.0/24` (Disponibilidade `us-east-1a`)
     - **Sub-rede privada**: `10.0.2.0/24` (Disponibilidade `us-east-1b`)

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

6. **Criar um Security Group**  
   - Vá até **Security Groups** > **Criar Security Group**.
   - Nomeie como `wordpress-sg`.
   - Adicione as seguintes regras:
     - **HTTP (80)**: Acesso de qualquer lugar (`0.0.0.0/0`)
     - **HTTPS (443)**: Acesso de qualquer lugar (`0.0.0.0/0`)
     - **SSH (22)**: Acesso restrito ao seu IP
     - **MySQL/Aurora (3306)**: Apenas para a VPC

---

## Etapa 2: Criar o Banco de Dados RDS

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

## Etapa 3: Criar o EFS para Armazenamento Compartilhado

1. **Acesse o console do EFS**.
2. Clique em **Criar sistema de arquivos**.
3. Selecione a VPC criada anteriormente.
4. Defina um nome (`wordpress-efs`).
5. Escolha as sub-redes privadas.
6. No Security Group, selecione `wordpress-sg`.
7. Finalize a criação e copie o **ID do EFS**.

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

## Etapa 5: Criar o Load Balancer

1. **Acesse o console do EC2**.
2. Vá até **Load Balancers** e clique em **Criar Load Balancer**.
3. Escolha **Application Load Balancer**.
4. Defina um nome (`wordpress-lb`).
5. Escolha **Internet-facing**.
6. Associe às sub-redes públicas.
7. Em **Security Groups**, selecione `wordpress-sg`.
8. Configure um **Target Group** com as instâncias EC2.
9. Finalize e copie o **DNS Name** do Load Balancer.

---

## Etapa 6: Configurar Auto Scaling Group

1. **Acesse o console do EC2**.
2. Vá até **Auto Scaling Groups** > **Criar Auto Scaling Group**.
3. Defina um nome (`wordpress-asg`).
4. Escolha o modelo de instância da Etapa 4.
5. Associe à **sub-rede pública**.
6. Configure regras de escala conforme necessidade.
7. Associe ao Load Balancer.
8. Finalize a configuração.

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
