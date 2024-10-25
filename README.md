Всем привет! Одна из моих рутинных задач - это подъем новых проектов и микросервисов в облаках. Для этого практически всегда нужны домены и поддомены с наличием SSL сертификата. У меня выработался подход, с помощью которого я автоматизировал процесс выдачи сертификатов с помощью certbot. О чём и хочу рассказать.

Почти все проекты, над которыми я работал или работаю, используют Nginx как HTTP-сервер и прокси. Также везде используется Docker Compose. Следовательно, я использую certbot в связке с ними.

Кстати, для мониторинга доступности Nginx сайтов и истечения срока SSL сертификатов я использую https://proverator.ru/ (мой проект). Когда ваш сайт падает, он присылает уведомления в Telegram.

Содержание

Создаём базовый Nginx-конфиг с доступом к certbot и заглушки для сертификатов

Пишем Dockerfile для certbot'a и скрипт генерации сертификата

Создаём docker-compose.yml и задаём переменные среды

Устанавливаем Docker

Запускаем Nginx и выдаём себе первый SSL сертификат

Запускаем Nginx с SSL

Настраиваем cron на обновление сертификата

Заключение

Дисклеймер: я не сис. админ и могу не знать тонкости настройки Nginx. Возможно даже что-то делаю не так. Но я рассказываю про свой опыт, который пригодился при запуске крупных коммерческих проектов.

Весь код есть на GitHub - https://github.com/RostislavDugin/certbot-nginx-docker/

Структура проекта такая:

Далее, по шагам, как выдавать себе сертификат.

1. Создаём базовый Nginx-конфиг с доступом к certbot и заглушки для сертификатов
   Для начала мы создаём nginx.conf в папке nginx. Указываем локации нашего сайта и certbot (certbot начнёт слушать входящие подключения при запуске в Docker'e). Это необходимо для выдачи сертификата.

# nginx.conf

worker_processes auto;

events {
}

http {
server {
listen 80;

    		location / {
    			# здесь нужно указать локальный адрес вашего
    			# сайта. У меня он в Docker'e на порту 3000. У
    			# вас может быть адрес в духе http://127.0.0.1:ПОРТ
    			proxy_pass http://172.17.0.1:3000;
    		}

    		# URL certbot'a, где он будет слушать входящие
    		# подключения во время выдачи SSL
    		location /.well-known {
    				# адрес certbot'a в Docker Compose на Linux
    				proxy_pass http://172.17.0.1:6000;
    		}
    }

}
Далее в той же папке /nginx создайте файлы cert.pem и key.pem.

Важно: напишите в эти файлы любой текст (например, "temp"). Docker Compose'y и certbot'y нужны заглушки файлов с любым содержанием, чтобы перезаписать их на настоящий ключ и сертификат.

2. Пишем Dockerfile для certbot'a и скрипт генерации сертификата
   Сначала в папке certbot создаём bash скрипт с названием generate-certificate.sh, который выдаст нам сертификат:

#!/bin/bash

# generate-certificate.sh

# чистим папку, где могут находиться старые сертификаты

rm -rf /etc/letsencrypt/live/certfolder\*

# выдаем себе сертификат (обратите внимание на переменные среды)

certbot certonly --standalone --email $DOMAIN_EMAIL -d $DOMAIN_URL --cert-name=certfolder --key-type rsa --agree-tos

# удаляем старые сертификаты из примонтированной

# через Docker Compose папки Nginx

rm -rf /etc/nginx/cert.pem
rm -rf /etc/nginx/key.pem

# копируем сертификаты из образа certbot в папку Nginx

cp /etc/letsencrypt/live/certfolder*/fullchain.pem /etc/nginx/cert.pem
cp /etc/letsencrypt/live/certfolder*/privkey.pem /etc/nginx/key.pem
Далее создаём Dockerfile для запуска certbot:

# Dockerfile

FROM ubuntu:22.04

EXPOSE 6000 80

# читаем переменные среды из .env

ARG DOMAIN_EMAIL
ARG DOMAIN_URL

# устанавливаем переменные среды в переменные

ENV DOMAIN_EMAIL=$DOMAIN_EMAIL
ENV DOMAIN_URL=$DOMAIN_URL

WORKDIR /certbot
COPY . /certbot
WORKDIR /certbot

RUN apt-get update
RUN apt-get -y install certbot

# запускаем скрипт генерации

CMD ["sh", "generate-certificate.sh"] 3. Создаём docker-compose.yml и задаём переменные среды
В корневой папке создаём docker-compose.yml со следующим содержанием:

# docker-compose.yml

version: "3"

services:
nginx:
image: nginx:1.23.3 # монтируем директорию nginx и сертификат с ключом
volumes: - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro - ./nginx/cert.pem:/etc/cert.pem - ./nginx/key.pem:/etc/key.pem
ports: - "80:80" - "443:443"

certbot:
ports: - "6000:80"
env_file: - .env # и снова мониторуем директорию nginx
volumes: - ./nginx/:/etc/nginx/
build:
context: ./certbot
dockerfile: Dockerfile # задаем переменные среды
args:
DOMAIN_EMAIL: ${DOMAIN_EMAIL}
DOMAIN_URL: ${DOMAIN_URL}
В той же корневой папке задаём переменные среды в файле .env. В нём прописываем домен и почту того, кто выпускает сертификат:

# .env

DOMAIN_URL=yourdomain.com
DOMAIN_EMAIL=youremail@mail.com 4. Устанавливаем Docker
Я использую скрипт ниже, чтобы одной командой скачать и установить Docker + Docker Compose. Можете установить своими силами или скопировать скрипт и вызвать через "sh install-docker.sh"

#!/bin/bash

# install-docker.sh

apt-get remove docker docker-engine docker.io containerd runc
apt-get install ca-certificates curl gnupg lsb-release
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin 5. Запускаем Nginx и выдаём себе первый SSL сертификат
Теперь, когда у нас всё готово, запускаем Nginx и выдаём себе сертификат.

# запуск в detach режиме

> docker compose up nginx --build -d

# запускаем без detach на случай, если допустили ошибку

> docker compose up certbot --build
> На экране должно появится похожее сообщение:

Главное - строчка Successfully received certificate. Это означает, что теперь файлы cert.pem и key.pem находятся в папке nginx с данными сертификата и ключа.

В случае, если что-то не так: старайтесь не запускать много раз выдачу скрипта с ошибкой и не забывайте про опцию --build, когда меняете файлы. Certbot даёт выпускать всего несколько сертификатов раз в несколько дней. Переборщите - попадёте в бан на несколько дней.

Один раз я так попал в бан, забыв обновить сертификат. На проде. В середине рабочего дня. Пришлось очень спешно покупать платный SSL и устанавливать вручную =).

6. Запускаем Nginx с SSL
   Пришло время запустить Nginx с SSL. Для этого обновляем файл nginx.conf в папке nginx:

# nginx.conf

worker_processes auto;

events {
}

http {
server {
listen 80;

    			# делаем переадресацию с HTTP на HTTPS
                location / {
                        return 301 https://$host$request_uri;
                }

    			# URL certbot'a, где он будет слушать входящие
    			# подключения во время выдачи SSL
                location /.well-known {
                        proxy_pass http://172.17.0.1:6000;
                }
        }

        server {
                listen       443 ssl http2;

    			# мы уже примонтировали сертификаты в Docker Compose
                ssl_certificate     /etc/cert.pem;
                ssl_certificate_key /etc/key.pem;

    			# здесь нужно указать локальный адрес к вашему
    			# сайту. У меня он в Docker'e на порту 3000. У
    			# вам может быть адрес http://127.0.0.1:ПОРТ
                location / {
                        proxy_pass http://172.17.0.1:3000;
                }
        }

}
И перезапускаем Nginx:

# запуск в detach режиме

> docker compose up nginx --build -d 7. Настраиваем cron на обновление сертификата
> Сертификат выдаётся на 3 месяца. Поэтому настраиваем обновление сертификата раз в определённый период. Я делаю обновление раз в месяц (т.к. опции cron не сильно очевидны, а @monthly понятно для любого, кто зайдёт в cron).

# запускаем cron

crontab -e

# указываем команду для обновления сертификата и перезапуска Nginx

@monthly cd /home/you_app_dir/ && docker compose up certbot --build && docker compose up nginx --build -d
Готово, сертификат будет обновляться раз в месяц.

Заключение
Такой подход я использую во всех своих проектах. Если у вас есть замечания или рекомендации, буду раз вашим комментариям!

Кстати, я разработал мониторинг сайтов. Если сайт упал, он шлёт уведомление в Telegram. И умеет заранее предупреждать об истечении SSL сертификата.

Если вам нужен - загуглите "Проверятор" или посмотрите ссылку в шапке моего профиля.
