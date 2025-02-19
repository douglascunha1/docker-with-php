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