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

```docker-compose
services: # Container 
  web: # Serviço(Container para o nginx)
    image: nginx:latest # Imagem a ser usada
    ports: # Porta a ser usada(host:container)
      - "80:80"

  app: # Serviço(Container para o php)
    build: # Builda uma imagem personalizada
      dockerfile: ./php/Dockerfile # Caminho do arquivo Dockerfile com a imagem personalizada

```