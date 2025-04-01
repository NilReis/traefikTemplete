# Traefik Local SSL Setup Guide

Este guia mostra como configurar o Traefik com SSL local para desenvolvimento.

## Pré-requisitos

1. Docker e Docker Compose instalados
2. mkcert instalado
   ```bash
   # No Linux (Ubuntu/Debian)
   sudo apt install mkcert
   
   # No macOS
   brew install mkcert
   ```

## Passos de Configuração

### 1. Estrutura de Diretórios

Primeiro, crie a seguinte estrutura de diretórios:
```bash
mkdir -p {certs,config,client,api}
```

### 2. Configurar SSL com mkcert

```bash
# Instalar CA local
mkcert -install

# Gerar certificados para os domínios
mkcert -key-file certs/local-key.pem -cert-file certs/local.pem testclinete.heroapp.tech testapi.heroapp.tech
```

### 3. Configurar Hosts

Adicione as seguintes linhas ao seu arquivo `/etc/hosts`:
```
127.0.0.1 testclinete.heroapp.tech
127.0.0.1 testapi.heroapp.tech
```

### 4. Arquivos de Configuração

#### docker-compose.yml
```yaml
services:
  traefik:
    image: traefik:v3.0
    container_name: traefik_local
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080" # Dashboard
    command:
      - "--log.level=DEBUG"
      - "--accesslog=true"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./config:/etc/traefik/config:ro
      - ./certs:/etc/traefik/certs:ro
    networks:
      - traefik_proxy

  client:
    image: nginx:alpine
    container_name: client
    volumes:
      - ./client:/usr/share/nginx/html
    expose:
      - "80"
    networks:
      - traefik_proxy

  api:
    image: nginx:alpine
    container_name: api
    volumes:
      - ./api:/usr/share/nginx/html
    expose:
      - "80"
    networks:
      - traefik_proxy

networks:
  traefik_proxy:
    name: traefik_proxy
```

#### traefik.yml (Configuração Estática)
```yaml
# API and Dashboard configuration
api:
  dashboard: true
  insecure: true # For local development ONLY

# EntryPoints definition
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https

  websecure:
    address: ":443"

# Providers configuration
providers:
  file:
    directory: "/etc/traefik/config"
    watch: true
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: traefik_proxy
```

#### config/dynamic.yml (Configuração Dinâmica)
```yaml
tls:
  certificates:
    - certFile: "/etc/traefik/certs/local.pem"
      keyFile: "/etc/traefik/certs/local-key.pem"

http:
  routers:
    client:
      entryPoints:
        - "websecure"
      rule: "Host(`testclinete.heroapp.tech`)"
      service: "client"
      tls: {}

    api:
      entryPoints:
        - "websecure"
      rule: "Host(`testapi.heroapp.tech`)"
      service: "api"
      tls: {}

  services:
    client:
      loadBalancer:
        servers:
          - url: "http://client:80"
    
    api:
      loadBalancer:
        servers:
          - url: "http://api:80"
```

### 5. Arquivos de Teste

#### client/index.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Test Client</title>
</head>
<body>
    <h1>Welcome to Test Client</h1>
    <p>This is testclinete.heroapp.tech</p>
</body>
</html>
```

#### api/index.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Test API</title>
</head>
<body>
    <h1>Welcome to Test API</h1>
    <p>This is testapi.heroapp.tech</p>
</body>
</html>
```

### 6. Iniciar os Serviços

```bash
# Iniciar todos os serviços
docker compose up -d

# Verificar os logs
docker compose logs -f
```

### 7. Testar os Endpoints

Após iniciar os serviços, você pode acessar:

- Dashboard Traefik: http://localhost:8080
- Site Cliente: https://testclinete.heroapp.tech
- API: https://testapi.heroapp.tech

## Recursos Adicionais

- O certificado SSL será válido até 1 de julho de 2027
- Redirecionamento automático de HTTP para HTTPS
- Dashboard Traefik para monitoramento
- Hot-reload das configurações do Traefik

## Solução de Problemas

1. Se os sites não carregarem:
   - Verifique se os domínios estão no arquivo hosts
   - Verifique se os certificados foram gerados corretamente
   - Verifique os logs do Traefik: `docker compose logs traefik`

2. Se o SSL não funcionar:
   - Verifique se o mkcert foi instalado corretamente
   - Verifique se os certificados estão no diretório correto
   - Execute `mkcert -install` novamente

3. Para reiniciar tudo do zero:
   ```bash
   docker compose down
   rm -rf certs/local*.pem
   mkcert -install
   mkcert -key-file certs/local-key.pem -cert-file certs/local.pem testclinete.heroapp.tech testapi.heroapp.tech
   docker compose up -d
