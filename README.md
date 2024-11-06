# traefik-casdoor-auth-webhook

Optmized [webhook](https://github.com/casdoor/traefik-casdoor-auth#start-the-webhook) image for the [treafik casedoor auth plugin](https://github.com/casdoor/traefik-casdoor-auth).

Either mount the [plugin config](https://github.com/casdoor/traefik-casdoor-auth#223-webhook-configuration-fileconfpluginjson) to /config/plugin.json or set the config with env variables:

```
CASDOOR_ENDPOINT
CASDOOR_CLIENT_ID
CASDOOR_CLIENT_SECRET
CASDOOR_ORGANIZATION
CASDOOR_APPLICATION
PLUGIN_ENDPOINT
```

Compose example:

```
version: '3'
services:
  webhook:
    image: baibaomen/traefik-casdoor-auth:latest
    ports:
      - 9999:9999
    restart: unless-stopped
    environment:
      CASDOOR_ENDPOINT: ''
      CASDOOR_CLIENT_ID: ''
      CASDOOR_CLIENT_SECRET: ''
      CASDOOR_ORGANIZATION: ''
      CASDOOR_APPLICATION: ''
      PLUGIN_ENDPOINT: ''
```


一个集成traefik和casdoor的示例配置：

```
(base) root@AlienJack:~/xinference# cat docker-compose.yml
version: "3.3"

services:
  xinference:
    image: xprobe/xinference:latest
    container_name: xinference
    command: xinference-local -H 0.0.0.0
    environment:
      - XINFERENCE_HOME=/data
    volumes:
      - ./data:/data
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    labels:
      - "traefik.enable=true"
      # HTTPS 配置
      - "traefik.http.routers.xinference.rule=Host(`xinference.baibaomen.com`)"
      - "traefik.http.routers.xinference.entrypoints=websecure"
      - "traefik.http.routers.xinference.tls.certresolver=myresolver"
      # HTTP 配置（重定向到 HTTPS）
      - "traefik.http.routers.xinference-http.rule=Host(`xinference.baibaomen.com`)"
      - "traefik.http.routers.xinference-http.entrypoints=web"
      - "traefik.http.routers.xinference-http.middlewares=redirect-to-https"
      # 服务端口配置
      - "traefik.http.services.xinference.loadbalancer.server.port=9997"
      # 认证中间件配置
      - "traefik.http.routers.xinference.middlewares=xinference-auth@docker"
      - "traefik.http.middlewares.xinference-auth.forwardauth.address=http://xinference-auth:9999/auth"
      - "traefik.http.middlewares.xinference-auth.forwardauth.authResponseHeaders=X-Forwarded-User"
      - "traefik.http.middlewares.xinference-auth.forwardauth.authResponseHeadersRegex=^X-|Cookie|Authorization"
    networks:
      - traefik-net

  xinference-auth:
    image: baibaomen/traefik-casdoor-auth:latest
    container_name: xinference-auth
    environment:
      - CASDOOR_ENDPOINT=https://auth.baibaomen.com
      # 下面这些，需要在casdoor中创建对应的应用后把相关属性复制过来填入。返回地址需要填写PLUGIN_ENDPOINT的地址，并且末尾需要带上/callback
      - CASDOOR_CLIENT_ID=theclientided3cb58f22c7a
      - CASDOOR_CLIENT_SECRET=thesecret1d094229b9ed2cda5323b97761e00
      - CASDOOR_ORGANIZATION=baibaomen
      - CASDOOR_APPLICATION=xinference
      - PLUGIN_ENDPOINT=https://xinference.baibaomen.com
      - LOG_LEVEL=debug
    labels:
      - "traefik.enable=true"
      # Callback 路由配置
      - "traefik.http.routers.xinference-auth-callback.rule=Host(`xinference.baibaomen.com`) && (Path(`/callback`) || PathPrefix(`/auth`))"
      - "traefik.http.routers.xinference-auth-callback.entrypoints=websecure"
      - "traefik.http.routers.xinference-auth-callback.tls.certresolver=myresolver"
      - "traefik.http.routers.xinference-auth-callback.service=xinference-auth"
      # 服务配置
      - "traefik.http.services.xinference-auth.loadbalancer.server.port=9999"
    networks:
      - traefik-net

networks:
  traefik-net:
    external: true
```
