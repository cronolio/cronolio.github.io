---
layout: post
sitemap:
  lastmod: 2023-02-01
title: devops на минималках - docker swarm - caddy
descr: caddy - мощная, расширяемая платформа для обслуживания запросов к вашему сайту.
keywords: docker swarm, caddy
---

#### Вступление

Как пишет [документация](https://caddyserver.com/docs/), caddy - это мощная,
расширяемая платформа для обслуживания запросов к вашему сайту.
Другими словами модульный прокси-сервер. Написан на golang.
Для сборки совместно с другими модулями используется утилита `xcaddy`.

#### В дебри
Начнем сразу с Dockerfile:
```
ARG caddy_version
FROM caddy:${caddy_version}-builder-alpine AS builder

ARG docker_proxy_version
ARG certmagic_s3_version
ARG coraza_caddy_version

RUN xcaddy build \
    --with github.com/lucaslorentz/caddy-docker-proxy@${docker_proxy_version} \
    --with github.com/techknowlogick/certmagic-s3@${certmagic_s3_version}
#   --with github.com/corazawaf/coraza-caddy@${coraza_caddy_version}

FROM caddy:${caddy_version}-alpine

COPY --from=builder /usr/bin/caddy /usr/bin/caddy

ENTRYPOINT ["/usr/bin/caddy"]
CMD ["docker-proxy"]
```

Первая интресность тут 
[ARG до FROM](https://docs.docker.com/engine/reference/builder/#understand-how-arg-and-from-interact)
Простыми словами - переменную можно взять из аргумента к `docker build` и использовать её в `FROM`.

Вторая интерерсность состоит в том, что это сборка в несколько этапов (multi stage build).
В первом образе собирается (названный как `builder`), затем копируется необходимое в чистый образ. 

Добавлены расширения caddy
* для работы с docker - caddy-docker-proxy;
* для централизованного хранения сертификатов - certmagic-s3;
* для waf - на будующее.

docker-compose.yml файл использую для сборки. 
```
version: "2.4"

services:
  caddy-build:
    container_name: caddy-build
    build:
      context: .
      dockerfile: ./Dockerfile
      args:
        - caddy_version=2.6.2
        - docker_proxy_version=v2.8.1
        - certmagic_s3_version=v1.2.3
        - coraza_caddy_version=v1.2.1
```

Тут интересность в том, что версия compose файла `2.4`.
Всё просто, `2.x` для обычного докера, для `docker-compose`. `3.x` для `docker swarm`.
Другая интересность - в передаче переменных, версии программ, в докерфайл при сборке через `args`.

Cобрать образ можно командой `docker-compose build --no-cache`.
Получится образ `caddy_caddy-build`, которым мы и воспользуемся в файле для swarm:
```
version: "3.4"

services:
  caddy:
    image: caddy_caddy-build:latest
    deploy:
      placement:
        constraints:
          - node.role == manager
      replicas: 1
      restart_policy:
        condition: any
      resources:
        reservations:
          cpus: "0.1"
          memory: 200M
    ports:
      - target: 80
        published: 80
        protocol: tcp
        mode: host
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./Caddyfile:/config/Caddyfile
    environment:
      - CADDY_INGRESS_NETWORKS=public
      - CADDY_DOCKER_CADDYFILE_PATH=/config/Caddyfile
    #command: "sh -c 'while true; do sleep 3; done'"
    networks: 
      - public

networks:
  public:
    external: true
```

Caddyfile прост:
```
{
  auto_https off
}
```
Просто не всегда нужны автоматические сертификаты.

#### лейблы

Для работы по 80 порту необходимо указать после проксируемого домена `example.com:80`,
либо указать протокол `http://example.com`. Я больше за порт:
```
labels:
  caddy: example.com:80
  caddy.reverse_proxy: "{{upstreams 80}}"
```
