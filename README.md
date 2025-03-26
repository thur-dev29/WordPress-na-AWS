# 💻 Projeto: WordPress com Docker + RDS + EFS + Load Balancer na AWS

Esssa documentação descreve todos os passos que segui para criar uma infraestrutura completa e escalável para rodar o WordPress na AWS usando:

- EC2 com Amazon Linux 2023
- Docker + Docker Compose
- RDS com MySQL
- EFS para arquivos persistentes (wp-content)
- Load Balancer (ALB)
- VPC com sub-redes públicas e privadas

---

## 📀 Arquitetura da Solução

```
Usuário → ALB (Load Balancer)
           ↓
         EC2 (Docker com WordPress)
           ↓
         EFS (wp-content)
           ↘
          RDS (MySQL)
```

## ⚒️ Etapas

---

### 1. Criar a VPC e Sub-redes

**Na VPC:**

- CIDR: `10.0.0.0/16`

**Sub-redes:**

| Tipo    | Nome              | CIDR           | AZ           |
|---------|-------------------|----------------|--------------|
| Pública | public-subnet-1   | 10.0.1.0/24    | us-east-1a   |
| Pública | public-subnet-2   | 10.0.2.0/24    | us-east-1b   |
| Privada | private-subnet-1  | 10.0.3.0/24    | us-east-1a   |
| Privada | private-subnet-2  | 10.0.4.0/24    | us-east-1b   |

> 🔒 Crie tabela de rotas pública com Internet Gateway e associe à sub-rede pública.  
> 🔒 Sub-redes privadas não devem ter acesso direto à internet (a não ser por NAT Gateway, opcional).

---

### 2. Criar o Banco de Dados RDS (MySQL)

- Engine: MySQL 8.x
- Modo: Single-AZ
- Usuário: `admin`
- Senha: `********`
- Crie um **DB Subnet Group** com as duas subnets privadas
- Marque como **"sem acesso público"**
- Importante: o banco `wordpress` deve ser criado manualmente depois

---

### 3. Criar e Configurar o EFS

- Nome: `wordpress-efs`
- VPC: `wordpress-vpc`
- Crie Mount Targets em **ambas as subnets privadas**
- Permita acesso via NFS (porta 2049) do SG da EC2

---

### 4. Criar Instância EC2

- AMI: Amazon Linux 2023
- Tipo: `t2.micro`
- Sub-rede: `public-subnet`
- IP público habilitado
- SG da EC2: liberar portas `22`, `80`, `2049` (NFS)
- Anexe script `user_data.sh` para provisionar automaticamente

---

### 5. Script `user_data.sh`

```bash
#!/bin/bash
dnf update -y
dnf install -y docker nfs-utils

# Docker
systemctl start docker
systemctl enable docker

# Docker Compose
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Montar EFS
mkdir -p /mnt/efs
mount -t nfs4 -o nfsvers=4.1 fs-XXXXXXXX.efs.us-east-1.amazonaws.com:/ /mnt/efs

mkdir -p /mnt/efs/wordpress
mkdir -p /opt/wordpress
cd /opt/wordpress

# Docker Compose
cat <<EOF > docker-compose.yml
version: '3.3'
services:
  wordpress:
    image: wordpress:latest
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: wordpress-db.cfi4aaqw0iv1.us-east-1.rds.amazonaws.com:3306
      WORDPRESS_DB_USER: admin
      WORDPRESS_DB_PASSWORD: SUA_SENHA
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - /mnt/efs/wordpress:/var/www/html/wp-content
EOF

docker-compose up -d
```

---

### 6. Permissões entre EC2 e RDS (SGs)

- No **SG do RDS**, adicione a seguinte regra de entrada:

| Tipo             | Porta | Origem           |
|------------------|-------|------------------|
| MySQL/Aurora     | 3306  | SG da instância EC2 |

---

### 7. Criar o Banco `wordpress` no RDS

Acesse via SSH na EC2:

```bash
sudo dnf install -y mariadb105
```

Conecte:

```bash
mysql -h wordpress-db.cfi4aaqw0iv1.us-east-1.rds.amazonaws.com -u admin -p
```

Crie o banco:

```sql
CREATE DATABASE wordpress;
```

---

### 8. Load Balancer (ALB)

- Application Load Balancer
- Público
- Listener: HTTP (porta 80)
- Target Group: HTTP, Instance, Porta 80
- Adicione a instância EC2
- Verifique se está `healthy`

---

### 9. Acessar WordPress

- Pegue o DNS do Load Balancer
- Acesse no navegador: `http://wordpress-alb-xxxxxx.us-east-1.elb.amazonaws.com`
- Siga a instalação do WordPress normalmente

---
