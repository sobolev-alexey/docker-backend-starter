### Подготовка Docker-образа и docker-compose.yml

-   подготовить docker образ;

-   подготовить docker-compose;

-   развернуть на сервере [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) c [acme-companion](https://github.com/nginx-proxy/acme-companion);

-   запускать проект обычным `docker-compose up -d` и наслаждаться рабочим продуктом.

Да, это конечно не Kubernetes и не супер отказоустойчивая архитектура. Но такой подход позволяет на VPS на 400р в месяц запустить десяток подобных проектов для личного использования.

Основная идея сборки состоит из следущих этапов:

1.  Собрать `frontend` (html, js, css).

2.  Собрать `backend`.

3.  Подсунуть в `backend` файлы из frontend в папку `static`.

4.  Собрать nginx образ, который будет разбирать траффик на статику и логику.

#### Dockerfile с multi-stage build

В проекте я использую node 16 на базе образа alpine. Поэтому начинаем `Dockerfile` со строчек

```
FROM node:16-alpine as base-builder
WORKDIR /app
```

Для начала нужно собрать `frontend` --- подтянуть зависимости, собрать html, js, css.

```
FROM base-builder as build_fe
WORKDIR /app
COPY ./frontend/package.json ./frontend/yarn.lock* ./
RUN yarn install
ADD ./frontend ./
RUN yarn generate
```

По итогу в этом промежуточном образе у нас будет собранный `frontend` в папке `/app/dist`

Далее собираем `backend`

```
FROM base-builder as build_be
WORKDIR /app
COPY ./backend/package.json ./backend/yarn.lock* ./
RUN yarn install
ADD ./backend ./
RUN yarn build
```

И получаем промежуточный образ только с `backend`. Теперь осталось собрать воедино в следующий промежуточный образ, который будет на 3001 порту слушать все запросы:

```
FROM node:16-alpine as finalNode
WORKDIR /app
COPY --from=build_be /app /app
COPY --from=build_fe /app/dist /app/static
CMD yarn start
```

*Я до конца не определился в необходимости этого этапа и, честно говоря, его можно и не делать. У нас, в итоге, получается backend, который умеет также отдавать статику приложения --- то есть полностью самостоятельно рабочий docker‑образ с приложением, который может работать без nginx. Но именно в рамках текущей статьи эта возможность не используется*

Теперь осталось собрать ту часть, которая будет с `nginx`:

```
FROM nginx:alpine as finalNginx
WORKDIR /usr/share/nginx/html
RUN rm -rf ./*
COPY --from=finalNode /app/static .
COPY ./docker/nginx/conf.conf /etc/nginx/conf.d/default.conf
CMD ["nginx", "-g", "daemon off;"]
```

Также надо не забыть положить файл конфигурации `nginx` по указанном пути:

```
# docker/nginx/conf.conf
server {
  listen 80 default_server;
	root /usr/share/nginx/html;

  client_max_body_size 20M;

  location / {
    root   /usr/share/nginx/html;
    index  index.html index.htm;
    try_files $uri $uri/ /index.html;
  }

  location /api {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_pass http://node:3001;
  }
}
```

Теперь мы, в рамках одного `Dockerfile` получили полную сборку всего, что нужно для работы приложения.

[Итоговый ](https://github.com/wormsoft/nest-and-nuxt-starter-kit/blob/master/Dockerfile)`Dockerfile` можно посмотреть в github репозитории.

#### Подготовка docker-compose.yml

Как я писал выше, я запускаю подобные проекты на сервере, используя `nginx-proxy`. Так что, первым делом, в конце файла надо объявить сеть `reverse-proxy`, через которую будет идти подключение из внешнего мира к моему контейнеру с `nginx`.

```
version: "3.8"
services:
# тут будут сервисы
networks:
  reverse-proxy:
    external:
      name: reverse-proxy
  back:
    driver: bridge
```

Также я добавил сеть `back` --- это изолированная сеть, через которую между собой будут общаться `nginx` и `backend`.

Теперь опишем как мы будем запускать наш образ с той частью, которая отвечает за `backend`:

```
node:
  build:
    context: .
    target: finalNode
  networks:
    - back
  expose:
    - 3001
  restart: always
  environment:
    - APP_PORT=3001
  healthcheck:
    test: wget --no-verbose --tries=1 --spider <http://localhost:3001> || exit 1
    timeout: 3s
    interval: 3s
    retries: 10

```

По порядку о каждом параметре:

-   `build`

    -   `context` --- что будет являться текущей директорией при сборке `Dockerfile`;

    -   `target` --- какую часть multi‑stage build нужно запускать в этом месте. В данном случае мы указываем, что собирать нужно все до `finalNode`.

-   `networks`

    -   тут мы указываем только back, т.к. во внешний мир контейнер ходить не будет и нужен только доступ от `nginx` к этому контейнеру.

-   `expose`

    -   этот пункт открывает доступ другим контейнерам в сети по перечисленным портам. В данном случае мы сообщаем, что в сети back контейнеры могут подключаться на 3001 порт.

-   `restart: always`

    -   сообщаем, что этот контейнер надо перезапускать всегда. Даже после перезапуска сервера проект будет запущен;

    -   будет работать до тех пор пока не выключим его командой `docker-compose down`.

-   `environment`

    -   передача переменных окружения в сам процесс node;

    -   в нашем случае только указываем порт, на котором мы хотим, чтобы `backend` был запущен.

-   `healthcheck`

    -   прекрасный инструмент для контроля работоспособности контейнера;

    -   `test` --- команда, от которой мы ожидаем `exit-code = 0` (какие есть еще можно прочитать [здесь](https://en.wikipedia.org/wiki/Exit_status));

    -   `timeout` --- время, которое может выполняться команда. Если команда зависла на больший срок, то проверка считается не пройденой;

    -   `interval` --- с какой частотой стоит выполнять команду, чтобы быть уверенным, что контейнер работает;

    -   `retries` --- после скольких неудачных ответов сервер помечается «нерабочим».

Теперь добавим блок с запуском `nginx`:

```
nginx:
  build:
    context: .
    target: finalNginx
  networks:
    - reverse-proxy
    - back
  expose:
    - 80
  restart: always
  depends_on:
    node:
      condition: service_healthy
  environment:
    - VIRTUAL_HOST=${DOMAIN}
    - VIRTUAL_PORT=80
    - LETSENCRYPT_HOST=${DOMAIN}
    - LETSENCRYPT_EMAIL=test@test.ru
```

Подробнее:

-   `build` тоже самое, что в `node` сервисе, только указан другой `target`, т.к. нам нужно получить ту часть, которая связана с `nginx`;

-   `networks` тут теперь 2 сети:

    -   `reverse-proxy` сеть, через которую будет доступ от контейнера `nginx-proxy`;

    -   `back` та сеть, в которой есть контейнер `node` чтобы можно было пересылать запросы ему.

-   `expose` сообщаем всем в сетях, что в этот контейнер можно стучаться на 80 порт. Это нужно `nginx-proxy` для обработки запросов;

-   `restart` аналогично сервису `node`;

-   `depends_on` тут мы указываем от каких сервисов мы зависим:

    -   если это не указать, то nginx будет запускаться вместе с остальными и может получиться ситуация, в которой `node` еще не запущен, а `nginx` уже готов принимать запросы, что нехорошо;

    -   поэтому мы указываем, что зависит от `node` сервиса;

    -   зависимость можно считать удовлетворенной только когда сервис прошел свой `healthcheck` (как раз блок `condition`).

-   `environment`

    -   тут мы указываем переменные окружения, которые нужны для работы `nginx-proxy`:

        -   `VIRTUAL_HOST` название домена доступа к приложению;

        -   `VIRTUAL_PORT` порт, на котором запущено приложение в контейнере;

        -   `LETSENCRYPT_HOST` тот же самый домен но уже для создания https сертификата;

        -   `LETSENCRYPT_EMAIL` электронная почта, куда писать о том, что скоро сертификат будет просрочен.

    -   тут используется внешняя переменная окружения `${DOMAIN}` и она будет записаться из файла `.env` который будет лежать рядом с `docker-compose.yml` файлом (подробнее [тут](https://docs.docker.com/compose/environment-variables/env-file/)).

[Конечный ](https://github.com/wormsoft/nest-and-nuxt-starter-kit/blob/master/docker-compose.yml)вариант файла также находится в github репозитории.

#### Дополнительные моменты в подготовке окружения

В корне проекта я создал файл `.dockerignore` чтобы, во время сборки, не перекачивать в контекст лишнего:

```
#.dockerignore
.idea
.git
**/.nuxt
**/dist
**/.output
**/node_modules
**/.env
```

Также создал `.env.example` в качестве файла-примера:

```
DOMAIN=domain.de
```

### Запуск приложения

#### Подготовка сервера

Разумеется на сервере должен уже стоять Docker. Если нет, то установите его по официальной [инструкции](https://docs.docker.com/engine/install/).

Далее необходимо на сервере запустить `nginx-proxy` и лучше это делать в отдельном месте на том‑же сервере (инструкция [здесь](https://github.com/nginx-proxy/nginx-proxy#docker-compose), но если нужно, то напишите в комментариях и дополню эту инструкцию здесь).

#### Запуск самого приложения

Запускается все это приложение очень простым образом:

1.  Клонируем исходники.

2.  Прописываем `DOMAIN` в `.env` файл в корне проекта.

3.  Запускаем командой `docker-compose up -d`.

Одной командой этот запуск можно сделать следующей командой:

```
DOMAIN=domain.ru && echo DOMAIN=$DOMAIN > .env && docker-compose up -d --build
```

Важно: заменить `domain.de` на свой домен, который уже направлен на сервер, где мы запускаем сервис.

#### Обновление версии приложения

Если нужно обновить исходники до последней версии, то можно выполнить следующую команду:

```
git fetch && git reset --hard origin/master && docker-compose up -d --build
```

И проект обновится и запустит обновленную версию на домене.
