FROM mariadb:latest

COPY ./db-load-script.sql /docker-entrypoint-initdb.d/

RUN apt-get update

EXPOSE 3306
