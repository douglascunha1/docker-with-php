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
COPY ./php/conf.d/xdebug.ini /usr/local/etc/php/conf.d/xdebug.ini

# Instala o Xdebug
RUN set -eux; \
	install-php-extensions xdebug