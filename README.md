# Tutorial básico de como usar o Docker e Docker Compose para subir um servidor PHP usando NGINX, MYSQL e REDIS.

## Dockerfile

O arquivo Dockerfile é onde utilizamos para montar nossas imagens personalizadas. Inicialmente, usamos uma imagem base como visto logo abaixo:

```dockerfile
# Imagem base e minimalista(alpine) usando FPM para utilização com NGINX.
FROM php:8.1-fpm-alpine
```

## Docker Compose

Após termos nossas imagens "prontas para uso", podemos trabalhar com a ideia de montar uma espécie de estrutura que 
permita múltiplos containers de executarem ao mesmo tempo, podendo depender dos outros containers e utilizar configurações específicas.

O arquivo docker-compose.yml faz isso, nele é possível definir "Serviços", ou seja, nossos serviços serão os containers definidos abaixo
como é o caso do web(Container para o NGINX) e app(Container para o PHP).

```yml
services: # Container 
  web: # Serviço(Container para o nginx)
    image: nginx:latest # Imagem a ser usada
    ports: # Porta a ser usada(host:container)
      - "80:80"

  app: # Serviço(Container para o php)
    build: # Builda uma imagem personalizada
      dockerfile: ./php/Dockerfile # Caminho do arquivo Dockerfile com a imagem personalizada

```

## Volume Mounting

Volumes é uma forma de persistir e compartilhar dados entre containers e entre container e host. O foco principal de uso deles são:

1. Persistência de dados
   - Manter os dados mesmo após o container ser removido
   - Exemplos são banco de dados, uploads de usuários etc.
2. Desenvolvimento Local
   - Sincronizar código do host com container
   - Permitir alterações em tempo real
3. Tipos de volumes
   - Volume Nomeado:
   ```yml
    volumes:
      - mysql_data:/var/lib/mysql # Gerenciado pelo Docker 
   ```
   - Bind Mount:
   ```yml
    volumes:
      - ./config:/etc/nginx/conf.d # Diretório local -> container
   ```
   - Tmpfs Mount (apenas Linux):
   ```yml
    volumes:
      - type: tmpfs
        target: /tmp # Armazenamento em memória
    ```
   - Declaração de Volumes:
   ```yml
    services:
      db:
        volumes:
          - mysql_data:/var/lib/mysql
    
    volumes: # Declaração global
      mysql_data: # Volume nomeado
        driver: local # Driver do volume
    ```