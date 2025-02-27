# WhaTicket SAAS - Guia de Instalação

Sistema de gestão de tickets multiusuário baseado no WhatsApp com recursos SAAS.

## Recursos
- Multi-atendentes
- Multi-WhatsApp
- Gestão de filas
- Respostas rápidas
- Gestão de usuários
- Interface intuitiva
- Multiempresas (SAAS)

## Requisitos do Sistema
- Ubuntu 20.04 LTS (Recomendado)
- Node.js 16.x
- PostgreSQL 12 ou superior
- Redis
- Docker e Docker Compose

## Instalação em Produção

### 1. Preparação do Servidor

```bash
# Atualizar o sistema
sudo apt update && sudo apt upgrade -y

# Instalar dependências básicas
sudo apt install -y git curl wget nginx software-properties-common postgresql postgresql-contrib redis-server

# Instalar Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker ${USER}

# Instalar Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.15.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Instalar Node.js 16.x
curl -fsSL https://deb.nodesource.com/setup_16.x | sudo -E bash -
sudo apt-get install -y nodejs

# Instalar PM2
sudo npm install -g pm2
```

### 2. Configuração do PostgreSQL

```bash
# Acessar o PostgreSQL
sudo -u postgres psql

# Criar banco de dados e usuário
CREATE DATABASE whaticket;
CREATE USER whaticket WITH ENCRYPTED PASSWORD 'sua_senha_forte';
GRANT ALL PRIVILEGES ON DATABASE whaticket TO whaticket;
\q

# Ajustar configurações do PostgreSQL (opcional)
sudo nano /etc/postgresql/12/main/postgresql.conf
```

### 3. Configuração do Redis

```bash
# Editar configuração do Redis
sudo nano /etc/redis/redis.conf

# Adicionar senha ao Redis (encontre a linha #requirepass e descomente)
requirepass sua_senha_forte

# Reiniciar Redis
sudo systemctl restart redis
```

### 4. Configuração do Projeto

```bash
# Clonar o repositório
cd /opt
sudo git clone https://github.com/seu-usuario/waticket.git
sudo chown -R ${USER}:${USER} waticket
cd waticket

# Configurar Backend
cd backend
cp .env.example .env
nano .env

# Configurar as variáveis de ambiente no .env:
NODE_ENV=production
BACKEND_URL=https://api.seu-dominio.com
FRONTEND_URL=https://app.seu-dominio.com
PROXY_PORT=443
PORT=8080

DB_DIALECT=postgres
DB_HOST=localhost
DB_PORT=5432
DB_USER=whaticket
DB_PASS=sua_senha_forte
DB_NAME=whaticket

JWT_SECRET=seu_jwt_secret_muito_seguro
JWT_REFRESH_SECRET=seu_jwt_refresh_secret_muito_seguro

REDIS_URI=redis://:sua_senha_forte@127.0.0.1:6379
REDIS_OPT_LIMITER_MAX=1
REDIS_OPT_LIMITER_DURATION=3000

USER_LIMIT=10000

# Instalar dependências e build do backend
npm install
npm run build
npx sequelize db:migrate
npx sequelize db:seed:all

# Configurar Frontend
cd ../frontend
cp .env.example .env
nano .env

# Configurar variável do frontend
REACT_APP_BACKEND_URL=https://api.seu-dominio.com

# Instalar dependências e build do frontend
npm install
npm run build
```

### 5. Configuração do Nginx

```bash
# Criar configuração para o backend
sudo nano /etc/nginx/sites-available/whaticket-backend

# Adicionar configuração
server {
    server_name api.seu-dominio.com;
    
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}

# Criar configuração para o frontend
sudo nano /etc/nginx/sites-available/whaticket-frontend

# Adicionar configuração
server {
    server_name app.seu-dominio.com;
    
    root /opt/waticket/frontend/build;
    index index.html index.htm;
    
    location / {
        try_files $uri $uri/ /index.html;
    }
}

# Ativar os sites
sudo ln -s /etc/nginx/sites-available/whaticket-backend /etc/nginx/sites-enabled
sudo ln -s /etc/nginx/sites-available/whaticket-frontend /etc/nginx/sites-enabled

# Testar e reiniciar nginx
sudo nginx -t
sudo systemctl restart nginx
```

### 6. Configuração SSL (Certbot)

```bash
# Instalar Certbot
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# Obter certificados SSL
sudo certbot --nginx -d api.seu-dominio.com
sudo certbot --nginx -d app.seu-dominio.com
```

### 7. Iniciar Aplicação

```bash
# Iniciar backend
cd /opt/waticket/backend
pm2 start dist/server.js --name whaticket-backend

# Iniciar frontend (opcional, já que está servido pelo nginx)
cd /opt/waticket/frontend
pm2 serve build 3000 --name whaticket-frontend --spa

# Salvar configuração do PM2
pm2 startup ubuntu
pm2 save
```

## Atualizações

Para atualizar o sistema, crie um script de atualização:

```bash
nano /opt/waticket/update.sh
```

Adicione o seguinte conteúdo:

```bash
#!/bin/bash
echo "Atualizando WhaTicket..."

cd /opt/waticket
git pull

# Atualizar backend
cd backend
npm install
npm run build
npx sequelize db:migrate
pm2 restart whaticket-backend

# Atualizar frontend
cd ../frontend
npm install
npm run build
pm2 restart whaticket-frontend

echo "Atualização concluída!"
```

Tornar o script executável:
```bash
chmod +x /opt/waticket/update.sh
```

## Acesso Inicial

Após a instalação, acesse o sistema através do navegador:
- Frontend: https://app.seu-dominio.com
- Credenciais padrão: admin@whaticket.com / admin

**Importante:** Altere a senha padrão após o primeiro acesso!

## Solução de Problemas

### Logs do Backend
```bash
pm2 logs whaticket-backend
```

### Logs do Frontend
```bash
pm2 logs whaticket-frontend
```

### Logs do Nginx
```bash
sudo tail -f /var/log/nginx/error.log
```

### Reiniciar Serviços
```bash
# Reiniciar backend
pm2 restart whaticket-backend

# Reiniciar frontend
pm2 restart whaticket-frontend

# Reiniciar nginx
sudo systemctl restart nginx

# Reiniciar PostgreSQL
sudo systemctl restart postgresql

# Reiniciar Redis
sudo systemctl restart redis
```

## Suporte

Para suporte, abra uma issue no GitHub ou entre em contato através dos canais oficiais.

## Licença

Este projeto está licenciado sob a licença MIT.
