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
   
## Adicionando o MySQL no nosso Docker Compose

Para adicionar o MySQL no nosso arquivo docker-compose.yml, basta adicionar um novo serviço, nesse caso dei o nome de db
que contém a imagem do mysql na versão 8.4.4, um volume para persistência dos dados onde o docker irá definir uma estratégia
apropriada para isso, a porta padrão do mysql mapeada do host para o container, restart sempre que algum problema acontecer(a menos
que o container seja parado manualmente), e variáveis de ambiente que serão executadas assim que o container subir. Por fim, adicionamos
um volume fora do services, onde definimos o seu nome como sendo mysqldata e o docker irá utilizar alguma estratégia apropriada.

```yml
services:
  # NGINX
  web: # Serviço(Container para o nginx)
    image: nginx:latest # Imagem a ser usada
    ports: # Porta a ser usada(host:container)
      - "80:80"
    volumes: # Permite criar volumes mapeados do host:container
      - ./nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf

  # PHP
  app:
    build:
      dockerfile: ./php/Dockerfile
    volumes: # Cria o volume mapeado do host:container
      - ./app:/app

  # MYSQL
  db:
    image: mysql:8.4.4
    volumes: # Deixamos o Docker definir a melhor estratégia de persistência dos dados
      - mysqldata:/var/lib/mysql
    ports: # Porta padrão do MySQL (host:container)
      - "3306:3306"
    restart: unless-stopped # Reinicia caso algo dê errado ou até o container ser parado
    environment: # Variáveis de ambiente do MySQL que serão utilizadas quando o container for inicializado
      MYSQL_ROOT_PASSWORD: rootpassword # Senha do usuário root
      MYSQL_USER: myuser # Usuário "normal"
      MYSQL_PASSWORD: userpassword # Senha do usuário "normal"
      MYSQL_DATABASE: docker-php # Nome do banco a ser usado

volumes:
  mysqldata:
```

## Configurando o Composer do PHP

Para configurar o composer do php, basta configurar o Dockerfile com a imagem do php como descrito logo abaixo:

```dockerfile
FROM php:8.1-fpm-alpine

# Instala as extensões PDO e PDO_MYSQL para usarmos com o PHP
RUN docker-php-ext-install pdo pdo_mysql

# Usado para setar o o usuário como super user no container
ENV COMPOSER_ALLOW_SUPERUSER=1

# Obtendo o composer usando multi-stage build
# See https://docs.docker.com/build/building/multi-stage/
# Copia a imagem do composer 2.8.5 para o diretório onde fica os binários do composer e passa para o diretório
# padrão na nossa imagem
COPY --from=composer:2.8.5 /usr/bin/composer /usr/bin/composer


# Copia os arquivos composer.json e composer.lock após o composer install ser executado
# A vantagem dessa cópia é que permite o cache do docker, dessa forma o composer install só será executado quando
# o composer.json e composer.lock soferem alterações
# See https://medium.com/@softius/faster-docker-builds-with-composer-install-b4d2b15d0fff
COPY ./app/composer.* ./

# Realiza a instalação das dependências do composer com argumentos para otimizar o processo
# @See https://getcomposer.org/doc/00-intro.md
RUN composer install --prefer-dist --no-dev --no-scripts --no-progress --no-interaction

# Copia os arquivos da aplicação para o diretório de trabalho(/var/www/html)
COPY ./app .

# Executa o autoload do composer otimizado
RUN composer dump-autoload --optimize
```

Em seguida, alteramos o docker-file.yml e setamos dois volumes, o primeiro é para evitar sobrescrita do diretório vendor
o segundo copia os arquivos de app para o diretório do container /var/www/html.

O ideal é ter dois arquivos do docker compose, um para produção e outro para desenvolvimento local.

```yml
services:
  # NGINX
  web: # Serviço(Container para o nginx)
    image: nginx:latest # Imagem a ser usada
    ports: # Porta a ser usada(host:container)
      - "80:80"
    volumes: # Permite criar volumes mapeados do host:container
      - ./nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf

  # PHP
  app:
    build:
      dockerfile: ./php/Dockerfile
    volumes: # Cria o volume mapeado do host:container
      - /var/www/html/vendor # Protege o diretório vendor para que o comando logo abaixo que não haja sobrescrita
      - ./app:/var/www/html # /var/www/html é o diretório no nosso container

  # MYSQL
  db:
    image: mysql:8.4.4
    volumes: # Deixamos o Docker definir a melhor estratégia de persistência dos dados
      - mysqldata:/var/lib/mysql
    ports: # Porta padrão do MySQL (host:container)
      - "3306:3306"
    restart: unless-stopped # Reinicia caso algo dê errado ou até o container ser parado
    environment: # Variáveis de ambiente do MySQL que serão utilizadas quando o container for inicializado
      MYSQL_ROOT_PASSWORD: rootpassword # Senha do usuário root
      MYSQL_USER: myuser # Usuário "normal"
      MYSQL_PASSWORD: userpassword # Senha do usuário "normal"
      MYSQL_DATABASE: docker-php # Nome do banco a ser usado

volumes:
  mysqldata:
```

O arquivo composer.json foi configurado da seguinte forma(apenas para demonstração). O src é um sub-diretório de app.

```json
{
  "name": "douglas/app",
  "description": "Demo app using docker and php",
  "minimum-stability": "dev",
  "license": "MIT",
  "authors": [
    {
      "name": "Douglas",
      "email": "contact.dougcunha@gmail.com"
    }
  ],
  "require": {
    "php": "^8.0",
    "ext-pdo": "*",
    "ext-pdo_mysql": "*",
    "symfony/cache": "^6.1",
    "predis/predis": "^2.0"
  },
  "require-dev": {
      "phpunit/phpunit": "^9.5"
  },
  "autoload": {
      "psr-4": {
          "App\\": "src"
      }
  },
  "scripts": {
      "run-phpunit": "vendor/bin/phpunit"
  }
}
```

## Usando uma build para desenvolvimento

Para usar uma build de desenvolvimento(docker compose), basta criar um arquivo chamado docker-compose.dev.yml

Abaixo temos o arquivo docker-compose.yml que é usado para "produção" por exemplo. Note que no serviço do nginx, foi criado um
novo Dockerfile para personalizar a imagem conforme necessário.

```yml
services:
  # NGINX
  web: # Serviço(Container para o nginx)
    build:
      dockerfile: ./nginx/Dockerfile
    ports: # Porta a ser usada(host:container)
      - "80:80"

  # PHP
  app:
    build:
      dockerfile: ./php/Dockerfile
    volumes: # Cria o volume mapeado do host:container
      - /var/www/html/vendor # Protege o diretório vendor para que o comando logo abaixo que não haja sobrescrita
      - ./app:/var/www/html

  # MYSQL
  db:
    image: mysql:8.4.4
    volumes: # Deixamos o Docker definir a melhor estratégia de persistência dos dados
      - mysqldata:/var/lib/mysql
    restart: unless-stopped # Reinicia caso algo dê errado ou até o container ser parado
    environment: # Variáveis de ambiente do MySQL que serão utilizadas quando o container for inicializado
      MYSQL_ROOT_PASSWORD: rootpassword # Senha do usuário root
      MYSQL_USER: myuser # Usuário "normal"
      MYSQL_PASSWORD: userpassword # Senha do usuário "normal"
      MYSQL_DATABASE: docker-php # Nome do banco a ser usado

volumes:
  mysqldata:
```

Dockerfile do nginx

```dockerfile
FROM nginx:latest

# Copia as configurações do host para o container
COPY ./nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf

# Copia os arquivos do host para o container
COPY ./app/public /var/www/html/public
```

Arquivo docker-compose.dev.yml para desenvolvimento local

```yml
services:
  # NGINX
  web: # Serviço(Container para o nginx)
    image: nginx:latest # Imagem a ser usada
    ports: # Porta a ser usada(host:container)
      - "80:80"
    volumes: # Permite criar volumes mapeados do host:container
      - ./nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf

  # PHP
  app:
    build:
      dockerfile: ./php/Dockerfile

  # MYSQL
  db:
    image: mysql:8.4.4
    volumes: # Deixamos o Docker definir a melhor estratégia de persistência dos dados
      - mysqldata:/var/lib/mysql
    ports: # Porta padrão do MySQL (host:container)
      - "3306:3306"
    restart: unless-stopped # Reinicia caso algo dê errado ou até o container ser parado
    environment: # Variáveis de ambiente do MySQL que serão utilizadas quando o container for inicializado
      MYSQL_ROOT_PASSWORD: rootpassword # Senha do usuário root
      MYSQL_USER: myuser # Usuário "normal"
      MYSQL_PASSWORD: userpassword # Senha do usuário "normal"
      MYSQL_DATABASE: docker-php # Nome do banco a ser usado

volumes:
  mysqldata:
```

Para buildar e subir os containers do docker compose para desenvolvimento local, basta usar o comando abaixo:

```bash
docker compose -f docker-compose.dev.yml up -d --build
```

Dessa forma é possível ter o ambiente de desenvolvimento configurado adequadamente.

## Variáveis de ambiente

Para utilizar variáveis de ambiente, basta criar um arquivo .env na raiz do projeto, em seguida, usar a notação ${} no 
docker-compose.dev.yml para pegar os valores de cada chave no arquivo .env, como visto abaixo:

```dotenv
MYSQL_PORT: 3306
MYSQL_PASSWORD: userpassword
MYSQL_DATABASE: docker-php
MYSQL_USER: myuser
```

Uma observação importante aqui, estamos configurando as variáveis de ambiente para o PHP utilizar o MySQL, ou seja, o serviço
a ser alterado é o app. No caso do MYSQL_HOST, usamos db que é o nome do serviço do mysql.

```yml
services:
  # NGINX
  web: # Serviço(Container para o nginx)
    image: nginx:latest # Imagem a ser usada
    ports: # Porta a ser usada(host:container)
      - "80:80"
    volumes: # Permite criar volumes mapeados do host:container
      - ./nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf

  # PHP
  app:
    build:
      dockerfile: ./php/Dockerfile
    volumes: # Cria o volume mapeado do host:container
      - ./app:/var/www/html
    environment:
        MYSQL_HOST: db # Mesmo nome que o serviço abaixo
        MYSQL_PORT: ${MYSQL_PORT}
        MYSQL_PASSWORD: ${MYSQL_PASSWORD}
        MYSQL_DATABASE: ${MYSQL_DATABASE}
        MYSQL_USER: ${MYSQL_USER}

  # MYSQL
  db:
    image: mysql:8.4.4
    volumes: # Deixamos o Docker definir a melhor estratégia de persistência dos dados
      - mysqldata:/var/lib/mysql
    ports: # Porta padrão do MySQL (host:container)
      - "3306:3306"
    restart: unless-stopped # Reinicia caso algo dê errado ou até o container ser parado
    environment: # Variáveis de ambiente do MySQL que serão utilizadas quando o container for inicializado
      MYSQL_ROOT_PASSWORD: rootpassword # Senha do usuário root
      MYSQL_USER: myuser # Usuário "normal"
      MYSQL_PASSWORD: userpassword # Senha do usuário "normal"
      MYSQL_DATABASE: docker-php # Nome do banco a ser usado

volumes:
  mysqldata:
```

## Redis

Redis é um banco em memória que utiliza a estrutura chave-valor para armazenar os dados. O Redis é muito utilizado para
cache, sessões, filas de mensagens etc...

Para adicionar o Redis no nosso docker compose, basta seguir os passos abaixo:

Uma observação importante é que no momento só foi adicionado o redis no arquivo docker-compose.dev.yml.

```yml
services:
  # NGINX
  web: # Serviço(Container para o nginx)
    image: nginx:latest # Imagem a ser usada
    ports: # Porta a ser usada(host:container)
      - "80:80"
    volumes: # Permite criar volumes mapeados do host:container
      - ./nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf

  # PHP
  app:
    build:
      dockerfile: ./php/Dockerfile
    volumes: # Cria o volume mapeado do host:container
      - ./app:/var/www/html
    environment:
        MYSQL_HOST: db # Mesmo nome que o serviço abaixo
        MYSQL_PORT: ${MYSQL_PORT}
        MYSQL_PASSWORD: ${MYSQL_PASSWORD}
        MYSQL_DATABASE: ${MYSQL_DATABASE}
        MYSQL_USER: ${MYSQL_USER}
        REDIS_HOST: cache # Nome do serviço a ser usado como host
        REDIS_PORT: ${REDIS_PORT} # Porta do Redis

  # MYSQL
  db:
    image: mysql:8.4.4
    volumes: # Deixamos o Docker definir a melhor estratégia de persistência dos dados
      - mysqldata:/var/lib/mysql
    ports: # Porta padrão do MySQL (host:container)
      - "3306:3306"
    restart: unless-stopped # Reinicia caso algo dê errado ou até o container ser parado
    environment: # Variáveis de ambiente do MySQL que serão utilizadas quando o container for inicializado
      MYSQL_ROOT_PASSWORD: rootpassword # Senha do usuário root
      MYSQL_USER: myuser # Usuário "normal"
      MYSQL_PASSWORD: userpassword # Senha do usuário "normal"
      MYSQL_DATABASE: docker-php # Nome do banco a ser usado

  # REDIS
  cache:
    image: redis:latest

volumes:
  mysqldata:
```

## Configurando o Xdebug

Para configurar o xdebug, iremos utilizar algo chamado multi-stage builds.

Um Multi-stage Build permite usar múltiplos estágios (stages) em um único Dockerfile, onde cada estágio pode usar 
uma imagem base diferente. O objetivo principal é copiar apenas os artefatos necessários do estágio de build para 
a imagem final, deixando para trás todas as dependências de compilação.

Abaixo temos um exemplo onde usamos dois alias, um para app e outro para app_dev, onde cada alias
possui configurações distintas.

```dockerfile
# app = alias
FROM php:8.1-fpm-alpine as app

# Permite instalar extensões do PHP
COPY --from=mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/local/bin/

# Instala as extensões do PHP
# -eux = Sair em caso de erros, sair em variáveis não definidas, imprimir cada comando conforme ele é executado
RUN set -eux; \
    install-php-extensions pdo pdo_mysql;

# Instala as extensões PDO e PDO_MYSQL para usarmos com o PHP
# RUN docker-php-ext-install pdo pdo_mysql

# Usado para setar o o usuário como super user no container
ENV COMPOSER_ALLOW_SUPERUSER=1

# Obtendo o composer usando multi-stage build
# See https://docs.docker.com/build/building/multi-stage/
# Copia a imagem do composer 2.8.5 para o diretório onde fica os binários do composer e passa para o diretório
# padrão na nossa imagem
COPY --from=composer:2.8.5 /usr/bin/composer /usr/bin/composer


# Copia os arquivos composer.json e composer.lock após o composer install ser executado
# A vantagem dessa cópia é que permite o cache do docker, dessa forma o composer install só será executado quando
# o composer.json e composer.lock soferem alterações
# See https://medium.com/@softius/faster-docker-builds-with-composer-install-b4d2b15d0fff
COPY ./app/composer.* ./

# Realiza a instalação das dependências do composer com argumentos para otimizar o processo
# @See https://getcomposer.org/doc/00-intro.md
RUN composer install --prefer-dist --no-dev --no-scripts --no-progress --no-interaction

# Copia os arquivos da aplicação para o diretório de trabalho(/var/www/html)
COPY ./app .

# Executa o autoload do composer otimizado
RUN composer dump-autoload --optimize


# app_dev é outro alias, basicamente estamos contruindo uma imagem(app_dev) em cima de outra(app)
FROM app as app_dev

# Seta o modo do xdebug para off
ENV XDEBUG_MODE=off

# Copia os arquivos de configuração do xdebug locais para o container
COPY ./php/conf.d/xdebug.ini $PHP_INI_DIR/conf.d/

# Instala o Xdebug
RUN SET -eux; \
    install-php-extensions xdebug
```

Após isso, criamos o diretório conf.d dentro do diretório php e criamos um arquivo dentro do conf.d chamado xdebug.ini
para declararmos nossas configurações para desenvolvimento local.

```
# Informa ao docker como conectar ao host quando for usado em desenvolvimento
xdebug.client_host = 'host.docker.internal'
```

No arquivo docker-compose.dev.yml adicionamos XDEBUG_MODE e extra_hosts, sendo XDEBUG_MODE um valor que vem da env e caso o valor nao
esteja presente na env, seta por padrão off. Já o extra_hosts é para mapear o host do container no linux.

```yml
services:
  # NGINX
  web: # Serviço(Container para o nginx)
    image: nginx:latest # Imagem a ser usada
    ports: # Porta a ser usada(host:container)
      - "80:80"
    volumes: # Permite criar volumes mapeados do host:container
      - ./nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf

  # PHP
  app:
    build:
      dockerfile: ./php/Dockerfile
      target: app # Seta qual imagem usar
    volumes: # Cria o volume mapeado do host:container
      - ./app:/var/www/html
      - ./php/conf.d/xdebug.ini:/usr/local/etc/php/conf.d/xdebug.ini:ro # Mapeia o volume do host para o container. ro = read only
    environment:
        MYSQL_HOST: db # Mesmo nome que o serviço abaixo
        MYSQL_PORT: ${MYSQL_PORT}
        MYSQL_PASSWORD: ${MYSQL_PASSWORD}
        MYSQL_DATABASE: ${MYSQL_DATABASE}
        MYSQL_USER: ${MYSQL_USER}
        REDIS_HOST: cache # Nome do serviço a ser usado como host
        REDIS_PORT: ${REDIS_PORT}
        XDEBUG_MODE: "${XDEBUG_MODE:-off}" # Recebe o valor da env ou seta por padrão off, caso nao tenha valor na env
    extra_hosts:
      - host.docker.internal:host-gateway # É necessário garantir que o host.docker.internal está correto e definido no linux

  # MYSQL
  db:
    image: mysql:8.4.4
    volumes: # Deixamos o Docker definir a melhor estratégia de persistência dos dados
      - mysqldata:/var/lib/mysql
    ports: # Porta padrão do MySQL (host:container)
      - "3306:3306"
    restart: unless-stopped # Reinicia caso algo dê errado ou até o container ser parado
    environment: # Variáveis de ambiente do MySQL que serão utilizadas quando o container for inicializado
      MYSQL_ROOT_PASSWORD: rootpassword # Senha do usuário root
      MYSQL_USER: myuser # Usuário "normal"
      MYSQL_PASSWORD: userpassword # Senha do usuário "normal"
      MYSQL_DATABASE: docker-php # Nome do banco a ser usado

  # REDIS
  cache:
    image: redis:latest

volumes:
  mysqldata:
```

Na .env adicionamos a linha XDEBUG_MODE: debug para permitir o debugar a nossa aplicação localmente.

Outra forma de setar manualmente(docker-compose.dev.yml) o target seria criando uma env.local passando a chave BUILD_TARGET com o valor app_dev

```
target: "${BUILD_TARGET:-app}"# Seta qual imagem usar baseado na env ou no padrao(app)
```
