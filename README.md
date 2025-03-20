# 👻 WORDPRESS COM DOCKER NA AWS

## 😽 Esse projeto tem finalidade explicar detalhadamente como configurar um ambiente AWS para hospedar uma aplicação WORDPRESS utilizando docker, incluindo a intalação do docker em uma instância EC2 com ubuntu, configuração do RDS MySQL, integração com EFS e Load Balancer, além claro, do visionamento com git.

---

# 📌 1. Arquitetura do Projeto
A infraestrutura é baseada em:
✅ *EC2* (Servidor de aplicação rodando WordPress em container Docker)  
✅ *EFS* (Armazenamento persistente para os arquivos do WordPress)  
✅ *RDS MySQL* (Banco de dados gerenciado para o WordPress)  
✅ *ALB (Load Balancer)* (Distribuição de tráfego entre múltiplas instâncias) 

---

# 📌 2. Criando a Infraestrutura na AWS

### 1️⃣ Criando a VPC e Sub-redes
1. Acesse o *AWS Console* > *VPC* > *VPC and more* > *Create VPC*.  
2. Crie uma *VPC com CIDR 10.0.0.0/16*.  
3. Crie *duas sub-redes públicas e duas privadas*.
4. Zona de disponibilidade escolha de AZ   
5. Associe um *Internet Gateway* à VPC para acesso externo.
6. Para anexar sua VPC vá até *Actions* > *Attach to VPC* e selecione a sua VPC 
---

### 2️⃣ Criando a Instância EC2 para o WordPress Privada
1. Vá para *EC2 > Launch Instance*.  
2. Escolha **Ubuntu 24.04 LTS** como AMI.  
3. Tipo de instância: *t2.micro* (grátis no Free Tier).  
4. Selecione sua VPC e a subnet será a privada  
6. Faça o download da *chave `.pem`* para conectar via SSH.  
7. Clique em *Launch Instance*.

Para a acessar a instância privada você precirar criar um instância publica (Bastion Host)
1. Vá para *EC2 > Launch Instance*.  
2. Escolha *Ubuntu 24.04 LTS** como AMI.  
3. Tipo de instância: *t2.micro* (grátis no Free Tier).  
4. Selecione sua VPC e a subnet será a publica  
6. Coloque a mesma chave anterior
7. Habilite o "Auto-assing Public IP" para que ela tenha um IP público
8. E na parte de secuurity group coleque SSH (porta 22) para "My IP".  
9. Clique em *Launch Instance*.

### 3️⃣  Conectar ao Bastion Host
- Pegue o IP público do Bastion Host no console  da AWS
- No terminal, conecte-se ao Bastion
````
 ssh -i "~/aws/keyfagundes.pem" ubuntu@<IP_PUBLICO_BASTION>
````
Agora dentro do Bastion, conecte-se á instância privada:
````
ssh -i "~/aws/keyfagundes.pem" ubuntu@<IP_PRIVADO>
````

---

# 📌 3. Na sua maquina instale o Dokcer e dependências
```
export DEBIAN_FRONTEND=noninteractive
sudo apt update -y && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose git nfs-common mysql-server
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
```
---

# 📌 4. Configuração do EFS (Armazenamento Persistente)

### 1️⃣ Criando o EFS
No AWS Console, vá para EFS > Create File System.
Selecione a VPC correta e sub-redes privadas.
Configure Segurança e Permissões.
Anote o ID do EFS (fs-xxxxxxxxxx).

---
### 2️⃣ Montando o EFS na EC2
````
EFS_ID="fs-xxxxxxxxxx"
REGION="us-east-1"
sudo mkdir -p /mnt/efs/wordpress
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 $EFS_ID.efs.$REGION.amazonaws.com:/ /mnt/efs
echo "$EFS_ID.efs.$REGION.amazonaws.com:/ /mnt/efs nfs4 defaults,_netdev 0 0" | sudo tee -a /etc/fstab
````

---

# 📌 5. Configuração do Banco de Dados MySQL no RDS
- Vá para RDS > Create DataBase
- Escolha MySQL e configure:
    Username: admin
    Password: SUA_SENHA_SEGURA
    Habilite acesso público (para teste).
  - Finalize e copie o endpoint do Banco de Dados

--- 

#  📌 6. Criando o docker-compose.yml
Crie o Arquivo Docker-compose.yml:
````
version: '3.8'

services:
  wordpress:
    image: wordpress:latest
    restart: always
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: bancocayo.cn0q06k0uoyw.us-east-1.rds.amazonaws.com
      WORDPRESS_DB_USER: admin
      WORDPRESS_DB_PASSWORD: SUA_SENHA_SEGURA
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - /mnt/efs/wordpress:/var/www/html

volumes:
  wp-data:
````
Logo depois, inicie os containers
````
cd /mnt/efs/wordpress
docker-compose up -d
````

----

# 📌 7. Configuração do Load Balancer (ALB)

- No Console da AWS, vá para EC2 > Load Balancers > Create Load Balancer
- Escolha Application Load Balancer
- Configure Listeners para a porta 80 HTTP
- Associe as instâncias EC2 ao Load Balancer
- Teste acessando o DNS do Load Balancer no navegador

---

# 📌 8. Configuração do Repositório GitHub

### 1️⃣ Inicializando o repositório
````
git init
git add .
git commit -m "Deploy inicial do WordPress na AWS"
git remote add origin https://github.com/SEU_USUARIO/wordpress-aws.git
git push -u origin main
````

---

# 📌 9. Automação com user_data.sh

### 1️⃣ Para automatizar todo o processo na inicialização da EC2, insira este script em "Advanced Details" > "User Data" na AWS
````
#!/bin/bash
export DEBIAN_FRONTEND=noninteractive
apt update -y && apt upgrade -y
apt install -y docker.io docker-compose git nfs-common mysql-server
systemctl start docker
systemctl enable docker
usermod -aG docker ubuntu
EFS_ID="fs-xxxxxxxxxx"
REGION="us-east-1"
mkdir -p /mnt/efs/wordpress
mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 $EFS_ID.efs.$REGION.amazonaws.com:/ /mnt/efs
echo "$EFS_ID.efs.$REGION.amazonaws.com:/ /mnt/efs nfs4 defaults,_netdev 0 0" | tee -a /etc/fstab
cd /mnt/efs/wordpress
docker-compose up -d
````

---

# 📌 10. Acessando o WordPress

### Após tudo estar rodando, acesse o WordPress pelo navegador:
````
http://SEU_IP_PUBLICO
````
