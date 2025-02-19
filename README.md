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

## Configurando o arquivo default.conf

Para criar configurações personalizadas para o NGINX, criamos um diretório chamado nginx na raiz do projeto e dentro do diretório
criamos outro diretório chamado conf.d, que conterá um arquivo chamado default.conf.

```nginx configuration
server {
    listen 80; # Ouve na porta 80
    server_name localhost; # Nome do servidor
    root /app/public; # Path a ser usado por nossa aplicação(requisições irão para esse path)
    index index.php; # Arquivo de entrada

    # location = Define regras para URL's específicas
    # ~ = Indica que iremos usar regex
    # \.php$ = Captura qualquer URL que termine com um .php
    location ~ \.php$ {
        # Encaminha requisições PHP para o container app na porta 9000(Padrão do PHP-FPM) e app é o nome do serviço definido no docker-compose
        fastcgi_pass app:9000;
        # Define o arquivo index padrão para requisições PHP e é usado quando a URL termina em /
        fastcgi_index index.php;
        # Passa o método HTTP (GET, POST, etc) para o PHP. $request_method é uma variável do Nginx
        fastcgi_param REQUEST_METHOD $request_method;
        # Define o caminho completo do script PHP a ser executado. $document_root é o valor definido em root. $fastcgi_script_name é o nome do arquivo PHP requisitado
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        # Inclui arquivo com parâmetros FastCGI padrão. Contém configurações comuns necessárias para PHP
        include fastcgi_params;
    }

    # Define regras para todas as outras URLs.
    # Captura requisições que não correspondem a outros blocos location.
    location / {
        # Tenta encontrar arquivos nesta ordem:
            # 1. $uri: Tenta encontrar o arquivo exato solicitado
            # 2. $uri/: Tenta encontrar como diretório
            # 3. /index.php?$query_string: Se nada for encontrado, redireciona para index.php
        # $query_string mantém parâmetros GET da URL original
        try_files $uri $uri/ /index.php?$query_string;
    }
}
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