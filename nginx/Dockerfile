FROM nginx:latest

# Copia as configurações do host para o container
COPY ./nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf

# Copia os arquivos do host para o container
COPY ./app/public /var/www/html/public