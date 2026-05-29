# 后端服务部署文档

[toc]

## 文档概述

本文档用于说明智慧幕墙系统后端服务的部署流程、CI/CD 自动化工作流、Docker Compose 编排方式、服务端口映射关系以及常见 Docker 调试方法。

本文档适用于以下人员：

- 后端开发人员
- 运维部署人员
- 测试人员
- 项目维护人员

本文档覆盖内容包括：

- 后端服务部署环境准备
- Docker Compose 自动化部署
- GitHub Actions CI/CD 工作流
- 后端服务端口对应关系
- Nginx / API Gateway 转发说明
- Docker 容器调试与故障排查

## 后端部署总体架构

系统后端采用容器化部署方式，各后端服务通过 Docker 镜像进行打包，并使用 Docker Compose 统一编排、启动和管理。

整体部署结构如下：

用户请求首先进入 Nginx/API Gateway，由网关根据请求路径将流量转发到不同的后端服务。各后端服务运行在独立容器中，通过固定端口暴露服务能力。业务服务、算法服务、评估服务等模块相互独立，便于后续扩展、升级和故障隔离。

部署流程主要包括：

1. 开发人员提交代码到 GitHub 仓库；
2. GitHub Actions 触发 CI/CD 工作流；
3. 工作流完成代码拉取、依赖安装、镜像构建和镜像推送；
4. 服务器拉取最新镜像；
5. Docker Compose 根据配置文件重新创建或更新容器；
6. Nginx/API Gateway 将外部请求转发到对应后端服务。

## 部署环境要求

### 服务器环境

| 项目 | 要求 |
|---|---|
| 操作系统 | Ubuntu / Debian / CentOS 等 Linux 系统 |
| CPU | 2 核及以上 |
| 内存 | 2GB 及以上 |
| Docker | 建议使用 Docker Engine 最新稳定版 |
| Docker Compose | 建议使用 Docker Compose v2 |
| 网络 | 服务器需要能够访问 Docker Hub 或私有镜像仓库 |

### 基础软件

服务器需要提前安装以下组件：

- Docker
- Docker Compose
- Git
- Nginx
- curl
- vim / nano
- systemctl 相关服务管理工具

### 目录结构

建议在服务器中统一使用如下目录结构：

```bash
/home/ecs-user/Intelligent_Curtain_Wall/
├── docker-compose.yml
├── .env
├── nginx/
│   └── default.conf
├── logs/
├── scripts/
│   ├── deploy.sh
│   └── rollback.sh
└── services/
```

## 代理软件部署

### 代理服务位置

位于服务器8.159.143.133和8.153.161.229的ecs-user/clash-for-linux文件夹中

### 下载项目

下载项目

```bash
git clone https://github.com/wnlen/clash-for-linux.git
```

进入到项目目录，编辑`.env`文件，修改变量`CLASH_URL`的值。

```bash
cd clash-for-linux
vim .env
```

> **注意：** `.env` 文件中的变量 `CLASH_SECRET` 为自定义 Clash Secret，值为空时，脚本将自动生成随机字符串。

需要自己添加url订阅链接

```
# Clash 订阅地址
export CLASH_URL='更改为你的clash订阅地址'
export CLASH_SECRET=''
export CLASH_HEADERS='User-Agent: ClashforWindows/0.20.39'

# Clash 监听配置
export CLASH_HTTP_PORT=7890
export CLASH_SOCKS_PORT=7891
export CLASH_REDIR_PORT=7892
export CLASH_LISTEN_IP=0.0.0.0
export CLASH_ALLOW_LAN=true

# External Controller (RESTful API) 配置
export EXTERNAL_CONTROLLER_ENABLED=true
export EXTERNAL_CONTROLLER=0.0.0.0:9090
```

<br>

### 启动程序

直接运行脚本文件`start.sh`

- 进入项目目录

```bash
$ cd clash-for-linux
```

- 运行启动脚本

```bash
$ sudo bash start.sh

正在检测订阅地址...
Clash订阅地址可访问！                                      [  OK  ]

正在下载Clash配置文件...
配置文件config.yaml下载成功！                              [  OK  ]

正在启动Clash服务...
服务启动成功！                                             [  OK  ]

Clash Dashboard 访问地址：http://<ip>:9090/ui
Secret：xxxxxxxxxxxxx

请执行以下命令加载环境变量: source /etc/profile.d/clash.sh

请执行以下命令开启系统代理: proxy_on

若要临时关闭系统代理，请执行: proxy_off

```

```bash
$ source /etc/profile.d/clash.sh
$ proxy_on
```

- 检查服务端口

```bash
$ netstat -tln | grep -E '9090|789.'
tcp        0      0 127.0.0.1:9090          0.0.0.0:*               LISTEN     
tcp6       0      0 :::7890                 :::*                    LISTEN     
tcp6       0      0 :::7891                 :::*                    LISTEN     
tcp6       0      0 :::7892                 :::*                    LISTEN
```

- 检查环境变量

```bash
$ env | grep -E 'http_proxy|https_proxy'
http_proxy=http://127.0.0.1:7890
https_proxy=http://127.0.0.1:7890
```

以上步骤如果正常，说明服务clash程序启动成功，现在就可以体验高速下载github资源了。

<br>

### 重启程序

重新进行以上三步即可

```
sudo bash start.sh
source /etc/profile.d/clash.sh
proxy_on
```

### 切换节点

由于有些节点可能不可用，因此开发者新增了一个切换节点的功能

#### 创建start-and-switch.sh脚本

将以下代码复制进脚本中
```
#!/usr/bin/env bash
set -euo pipefail

CLASH_DIR="/home/ecs-user/clash-for-linux"
GROUP_NAME="🔰国外流量"
NODE_NAME="美国G | 0.1x-下载专用"

cd "$CLASH_DIR"

echo "[1/6] Restart Clash"
sudo bash shutdown.sh || true
sleep 2

START_LOG="$(mktemp)"
sudo bash start.sh | tee "$START_LOG"

echo "[2/6] Wait for Clash API"
for i in $(seq 1 30); do
  if ss -lntp | grep -q ':9090'; then
    echo "Clash API is ready"
    break
  fi

  if [ "$i" -eq 30 ]; then
    echo "Clash API did not become ready"
    exit 1
  fi

  sleep 1
done

echo "[3/6] Read Clash API secret"
SECRET="$(awk '/Secret:/ {print $2}' "$START_LOG" | tail -n1)"

if [ -z "$SECRET" ]; then
  echo "Failed to read Clash Secret from start.sh output"
  exit 1
fi

echo "[4/6] Switch node: $GROUP_NAME -> $NODE_NAME"

GROUP_NAME="$GROUP_NAME" NODE_NAME="$NODE_NAME" CLASH_SECRET="$SECRET" python3 - <<'PY'
import json
import os
import urllib.parse
import urllib.request

group = os.environ["GROUP_NAME"]
node = os.environ["NODE_NAME"]
secret = os.environ["CLASH_SECRET"]

url = "http://127.0.0.1:9090/proxies/" + urllib.parse.quote(group, safe="")
data = json.dumps({"name": node}).encode()

headers = {
    "Content-Type": "application/json",
    "Authorization": "Bearer " + secret,
}

req = urllib.request.Request(url, data=data, method="PUT", headers=headers)

# 访问 Clash 控制 API 时不要走系统代理，否则可能导致 401 或异常转发
opener = urllib.request.build_opener(urllib.request.ProxyHandler({}))

with opener.open(req, timeout=15) as resp:
    print("OK", resp.status)
PY

echo "[5/6] Enable shell proxy"
source /etc/profile.d/clash.sh
proxy_on

echo "[6/6] Test proxy"
curl -I --connect-timeout 15 -x http://127.0.0.1:7890 https://www.google.com

echo "[OK] Clash started, node switched, proxy enabled."

```

#### 运行脚本

```
sudo bash start-and-switch.sh
```

#### 测试
```
curl -I www.google.com
```
若出现“200 OK”标志即为成功

### 提示

#### 谨慎选择其他代理软件

由于不同代理软件所涉及的代理协议不同，有的代理协议会被阿里云服务器识别而禁止用户翻墙，所以选择正确的代理软件和代理协议是成功的第一步。

#### 选择合适的机场

由于云服务器IP极易被识别的特性和机场要防止被云服务器“二次中转”所造成的商业滥用，换句话说，怕用户拿服务器去给别人当中转机场，所以比较正规的梯子往往都会识别大型的服务器平台，如阿里云。因此使用一个限制宽松，不会特意识别服务器的机场是十分有必要的，建议多换几个同学的机场试试。除此之外，由于docker镜像往往比较大（可能达几G），有些机场还会限制单次下载的流量，所以在阿里云服务器上配置合适的机场并非易事。

## 后端服务与端口对应关系


系统中的后端服务通过 Docker 容器运行，每个服务在宿主机上暴露一个固定端口，Nginx/API Gateway 根据请求路径将流量转发到对应服务。

| 服务名称 | 容器名称 | 镜像名称 | 宿主机端口 | 容器端口 | 功能说明 |
|---|---|---|---:|---:|---|
| 用户认证服务 | user-authentication | clearpool/intelligent-curtain-wall:user-authentication-new | 8008 | 8000 | 用户登录、注册、权限认证 |
| 裂缝检测服务 | crack-detection | clearpool/intelligent-curtain-wall:crack-detection | 18001 | 8080 | 石材裂缝识别与检测 |
| 幕墙评估服务 | resilience-assessment | clearpool/intelligent-curtain-wall:resilience-assessment | 18005 | 8080 | 幕墙韧性与安全评估 |
| 污渍检测服务 | stain-detection | clearpool/intelligent-curtain-wall:stain-detection | 18007 | 8080 | 石材污渍识别与分析 |
| 振动检测服务 | vibration-detection | clearpool/intelligent-curtain-wall:vibration-detection | 18009 | 8080 | 幕墙振动数据分析 |

端口访问示例：


```bash
curl http://localhost:8008/api/account/login
curl http://localhost:18001/
curl http://localhost:18005/
curl http://localhost:18007/
curl http://localhost:18009/
```

说明：

- 宿主机端口用于服务器外部或 Nginx 访问；
- 容器端口为服务在容器内部监听的端口；
- 不建议直接暴露数据库端口到公网；
- 对外访问应优先通过 Nginx/API Gateway 统一入口。


---

## Docker Compose 自动化部署

系统使用 Docker Compose 对多个后端服务进行统一编排。通过一个 `docker-compose.yml` 文件，可以同时完成镜像拉取、容器启动、端口映射、环境变量注入和服务重启策略配置。

### 5.1 docker-compose.yml 示例

```yaml
services:
  user-authentication:
    image: clearpool/intelligent-curtain-wall:user-authentication-new
    container_name: user-authentication
    restart: always
    ports:
      - "8008:8000"
    env_file:
      - .env
    networks:
      - curtainwall-net

  crack-detection:
    image: clearpool/intelligent-curtain-wall:crack-detection
    container_name: crack-detection
    restart: always
    ports:
      - "18001:8080"
    volumes:
      - /home/ecs-user/Intelligent_Curtain_Wall/crack-detection:/segformerProject/model
    networks:
      - curtainwall-net

  stain-detection:
    image: clearpool/intelligent-curtain-wall:stain-detection
    container_name: stain-detection
    restart: always
    ports:
      - "18007:8080"
    networks:
      - curtainwall-net

  resilience-assessment:
    image: clearpool/intelligent-curtain-wall:resilience-assessment
    container_name: resilience-assessment
    restart: always
    ports:
      - "18005:8080"
    networks:
      - curtainwall-net

  vibration-detection:
    image: clearpool/intelligent-curtain-wall:vibration-detection
    container_name: vibration-detection
    restart: always
    ports:
      - "18009:8080"
    networks:
      - curtainwall-net

networks:
  curtainwall-net:
    driver: bridge
```

### 常用部署命令

启动全部服务：

```
docker compose up -d
```

停止全部服务：

```
docker compose down
```

拉取最新镜像：

```
docker compose pull
```

重新部署：

```
docker compose pull
docker compose up -d
```

查看运行状态：

```
docker compose ps
```



---

## CI/CD 工作流部署

系统使用 GitHub Actions 实现 CI/CD 自动化部署。开发人员将代码推送到指定分支后，GitHub Actions 自动执行构建、镜像推送和远程服务器部署流程。

### CI/CD 流程说明

CI/CD 工作流主要包括以下步骤：

1. 监听指定分支代码提交；
2. 拉取项目源代码；
3. 配置 Docker Build 环境；
4. 登录 Docker Hub 或私有镜像仓库；
5. 构建后端服务 Docker 镜像；
6. 推送镜像到镜像仓库；
7. 通过 SSH 连接远程服务器；
8. 在服务器上执行部署脚本；
9. 拉取最新镜像并重启 Docker Compose 服务。

### GitHub Actions 示例

```yaml
name: CI/CD Pipeline

on:
  workflow_call:
    inputs:
      image-tag:
        description: 'Docker image tag'
        required: true
        type: string
    secrets:
      WORKFLOW_PRIVATE_KEY:
        required: true

jobs:
  ci-cd-pipeline:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout the code
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
          ref: ${{ github.ref }}

      - name: Set up SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.WORKFLOW_PRIVATE_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan github.com >> ~/.ssh/known_hosts

      - name: Clone secrets repository
        run: |
          git clone git@github.com:Intelligent-Curtain-Wall/.secrets.git
          if [ ! -d ".secrets" ]; then
            echo "Error: .secrets directory not found!"
            exit 1
          fi

      - name: Log in to Docker Hub
        run: |
          echo "$(cat .secrets/DOCKER_PASSWORD)" | docker login --username "$(cat .secrets/DOCKER_USERNAME)" --password-stdin

      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: ./
          file: ./Dockerfile
          push: true
          tags: docker.io/clearpool/intelligent-curtain-wall:${{ inputs.image-tag }}
      



      - name: Connect to remote server and execute deployment script
        run: |
          mv .secrets/MATPOOL_SSH_PRIVATE_KEY ~/.ssh/matpool_id_rsa
          chmod 600 ~/.ssh/matpool_id_rsa

          SSH_PORT="$(cat .secrets/MATPOOL_SSH_PORT)"
          SSH_USER="$(cat .secrets/MATPOOL_SSH_USERNAME)"
          SSH_HOST="$(cat .secrets/MATPOOL_SSH_HOST)"

          ssh -i ~/.ssh/matpool_id_rsa \
            -p "$SSH_PORT" \
            -o StrictHostKeyChecking=accept-new \
            -o ConnectTimeout=20 \
            -o ServerAliveInterval=20 \
            -o ServerAliveCountMax=3 \
            "$SSH_USER@$SSH_HOST" 'bash -s' <<'REMOTE_SCRIPT'
          set -euo pipefail

          CLASH_DIR="/home/ecs-user/clash-for-linux"
          PROJECT_DIR="/home/ecs-user/Intelligent_Curtain_Wall"

          CLASH_GROUP="🔰国外流量"
          CLASH_NODE="美国G | 0.1x-下载专用"

          echo "[1/7] Restart Clash proxy"
          cd "$CLASH_DIR"
          sudo bash shutdown.sh || true
          sleep 2

          sudo bash start.sh | tee /tmp/clash-start.log

          echo "[2/7] Wait for Clash ports"
          for i in $(seq 1 30); do
            if ss -lntp | grep -q ':7890' && ss -lntp | grep -q ':9090'; then
              echo "Clash proxy and API are ready"
              break
            fi

            if [ "$i" -eq 30 ]; then
              echo "Clash did not become ready in time"
              tail -n 100 /tmp/clash-start.log || true
              exit 1
            fi

            sleep 1
          done

          echo "[3/7] Read Clash API secret"
          CLASH_SECRET="$(awk '/Secret:/ {print $2}' /tmp/clash-start.log | tail -n1)"

          if [ -z "$CLASH_SECRET" ]; then
            echo "Failed to read Clash Secret from startup log"
            exit 1
          fi

          echo "[4/7] Switch Clash to Global mode and select node"
          CLASH_GROUP="$CLASH_GROUP" CLASH_NODE="$CLASH_NODE" CLASH_SECRET="$CLASH_SECRET" python3 - <<'PY'
          import json
          import os
          import urllib.parse
          import urllib.request

          group = os.environ["CLASH_GROUP"]
          node = os.environ["CLASH_NODE"]
          secret = os.environ["CLASH_SECRET"]

          headers = {
              "Content-Type": "application/json",
              "Authorization": "Bearer " + secret,
          }

          opener = urllib.request.build_opener(urllib.request.ProxyHandler({}))

          def request(method, path, body=None):
              url = "http://127.0.0.1:9090" + path
              data = None if body is None else json.dumps(body).encode()
              req = urllib.request.Request(url, data=data, method=method, headers=headers)
              with opener.open(req, timeout=15) as resp:
                  raw = resp.read()
                  return json.loads(raw.decode()) if raw else None

          def switch(selector, target):
              path = "/proxies/" + urllib.parse.quote(selector, safe="")
              request("PUT", path, {"name": target})
              print(f"Switched: {selector} -> {target}")

          proxies = request("GET", "/proxies")
          proxies = proxies.get("proxies", proxies)

          if group not in proxies:
              raise SystemExit(f"Group not found: {group}")

          if node not in proxies[group].get("all", []):
              raise SystemExit(f"Node not found in {group}: {node}")

          switch(group, node)

          request("PATCH", "/configs", {"mode": "Global", "ipv6": False})
          print("Mode switched to Global, IPv6 disabled")

          if "GLOBAL" in proxies:
              global_all = proxies["GLOBAL"].get("all", [])
              if node in global_all:
                  switch("GLOBAL", node)
              elif group in global_all:
                  switch("GLOBAL", group)
              else:
                  raise SystemExit("GLOBAL cannot select the target node or group")
          else:
              print("GLOBAL selector not found, only mode was switched")
          PY

          echo "[5/7] Enable shell proxy"
          source /etc/profile.d/clash.sh
          proxy_on

          echo "[6/7] Test Docker registry through proxy"
          HTTP_CODE="$(curl -4 -sS -o /dev/null -w '%{http_code}' \
            --connect-timeout 20 \
            -x http://127.0.0.1:7890 \
            https://registry-1.docker.io/v2/)"

          if [ "$HTTP_CODE" != "401" ]; then
            echo "Docker registry proxy test failed, HTTP code: $HTTP_CODE"
            exit 1
          fi

          echo "Docker registry is reachable through Clash proxy"

          echo "[7/7] Start deployment script in background"
          cd "$PROJECT_DIR"
          chmod +x automated-deployment.sh

          nohup bash -lc '
            set -e
            source /etc/profile.d/clash.sh
            proxy_on
            cd /home/ecs-user/Intelligent_Curtain_Wall
            ./automated-deployment.sh
          ' >> "$PROJECT_DIR/deployment.log" 2>&1 &

          echo $! > "$PROJECT_DIR/deployment.pid"

          echo "Deployment script started in background"
          echo "PID: $(cat "$PROJECT_DIR/deployment.pid")"
          echo "Log file: $PROJECT_DIR/deployment.log"
          REMOTE_SCRIPT

          echo "Remote deployment command completed."
          echo "You can check the deployment logs here: http://110.42.214.164:9000/deployment-logs"
```

### Secret 配置说明

CI/CD 工作流中不应直接写入密码、密钥、服务器 IP 等敏感信息，应统一配置在 GitHub Secrets 中。

| Secret 名称     | 作用                    |
| --------------- | ----------------------- |
| DOCKER_USERNAME | Docker Hub 用户名       |
| DOCKER_PASSWORD | Docker Hub 密码或 Token |
| SERVER_HOST     | 服务器公网 IP           |
| SERVER_SSH_KEY  | SSH 私钥                |
| SERVER_PORT     | SSH 端口，可选          |



---

## 自动化部署脚本

建议把服务器上的部署命令封装成脚本，方便 GitHub Actions 调用。

为减少人工操作，服务器中提供 `deployment.sh` 脚本，用于自动完成镜像拉取、容器更新和无用镜像清理。

### ```deployment.sh``` 示例

```bash
#!/bin/bash
set -e
LOG_DIR="/home/ecs-user/Intelligent_Curtain_Wall/deployment-logs"

export TZ="Asia/Shanghai"

TIMESTAMP=$(date +"%Y%m%d-%H:%M")
START_TIME=$(date +"%Y-%m-%d %H:%M:%S")
START_SECONDS=$(date +%s)


mkdir -p "$LOG_DIR"

{
    cleanup() {
        echo "[CLEANUP] Disable proxy and stop Clash"

        if [ -f /etc/profile.d/clash.sh ]; then
            source /etc/profile.d/clash.sh || true
            proxy_off || true
        fi

        sudo pkill -f clash-linux-amd64 || true
    }

    trap cleanup EXIT

    echo "Execution Start Time: $START_TIME"

    cd /home/ecs-user/Intelligent_Curtain_Wall

    imagesbefore=$(sudo docker images -q --filter "dangling=true")
    if [ -n "$imagesbefore" ]; then
        echo "Removing dangling images..."
        sudo docker rmi $imagesbefore
    else
        echo "No dangling images to remove."
    fi

    sudo docker compose pull
    sudo docker compose down
    sudo docker compose up -d

    imagesafter=$(sudo docker images -q --filter "dangling=true")
    if [ -n "$imagesafter" ]; then
        echo "Removing dangling images..."
        sudo docker rmi $imagesafter
    else
        echo "No dangling images to remove."
    fi

    END_TIME=$(date +"%Y-%m-%d %H:%M:%S")
    END_SECONDS=$(date +%s)
    ELAPSED_TIME=$((END_SECONDS - START_SECONDS))

    echo "Execution End Time: $END_TIME"
    echo "Total Execution Time: $ELAPSED_TIME seconds"
} &> "$LOG_DIR/$TIMESTAMP.txt"

```

赋予执行权限：

```
chmod +x deployment.sh
```

手动执行：

```
./deployment.sh
```

---

## Nginx / API Gateway 转发说明

系统使用 Nginx 作为 API Gateway，对外提供统一访问入口。Nginx 根据不同路径将请求转发到对应后端服务，从而避免前端直接访问多个后端端口。

示例配置：

```nginx
server {
    listen 80;
    server_name _;

    location /api/account/ {
        proxy_pass http://127.0.0.1:8008;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/crack/ {
        proxy_pass http://127.0.0.1:18001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/stain/ {
        proxy_pass http://127.0.0.1:18007;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/resilience/ {
        proxy_pass http://127.0.0.1:18005;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/vibration/ {
        proxy_pass http://127.0.0.1:18009;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

修改 Nginx 配置后，需要执行：

```
sudo nginx -t
sudo systemctl reload nginx
```


这里需要特别注意：  
如果后端接口本身包含 `/api/account/login`，Nginx 不要随便把 `/api` 前缀删掉，否则可能导致后端路由匹配失败。

---

## Docker 调试与故障排查

### 查看容器状态

```bash
docker ps
docker ps -a
docker compose ps
```

### 查看容器日志

```
docker logs user-authentication
docker logs -f user-authentication
docker compose logs -f
```

### 进入容器内部

```
docker exec -it user-authentication bash
```

如果容器内没有 bash，可以使用：

```
docker exec -it user-authentication sh
```

### 测试宿主机端口

```
curl http://127.0.0.1:8008
curl http://127.0.0.1:18001
```

### 测试容器内部服务

```
docker exec -it user-authentication sh
curl http://127.0.0.1:8000
```

### 查看端口占用

```
sudo ss -tulpn
sudo ss -tulpn | grep 8008
```

### 查看容器详细信息

```
docker inspect user-authentication
```

### 常见问题

| 问题                      | 可能原因                   | 解决方法                        |
| ------------------------- | -------------------------- | ------------------------------- |
| 502 Bad Gateway           | Nginx 无法连接后端服务     | 检查容器是否运行、端口是否正确  |
| 500 Internal Server Error | 后端程序内部异常           | 查看后端日志                    |
| 401 Unauthorized          | 鉴权失败                   | 检查 Token、账号密码、认证逻辑  |
| 403 Forbidden             | 权限不足或接口被拦截       | 检查 Spring Security / 权限配置 |
| 端口无法访问              | 防火墙或端口未监听         | 检查 `ss -tulpn` 和安全组       |
| 镜像拉取失败              | 网络无法访问 Docker Hub    | 配置镜像源或代理                |
| 容器反复重启              | 程序启动失败或环境变量错误 | 查看 `docker logs`              |


---

## 回滚方案

当新版本部署后出现严重故障时，可以通过镜像版本回滚恢复服务。

### 使用旧版本镜像回滚

修改 `docker-compose.yml` 中的镜像标签，例如：

```yaml
image: clearpool/intelligent-curtain-wall:user-authentication-v1.0.0
```

然后执行：

```
docker compose pull
docker compose up -d
```

### 使用 Git 回滚配置文件

```
git log --oneline
git checkout <commit_id> docker-compose.yml
docker compose up -d
```

### 回滚注意事项

- 回滚前需要确认数据库结构是否兼容；
- 回滚时应保留当前故障日志；
- 回滚完成后需要重新测试核心接口；
- 不建议直接删除旧镜像，应至少保留最近一个稳定版本。



---

## 安全与维护建议
为保证后端服务稳定运行，部署过程中应遵循以下原则：

1. 不在代码仓库中明文存储数据库密码、Token、SSH 私钥等敏感信息；
2. 使用 GitHub Secrets 管理 CI/CD 所需密钥；
3. 数据库端口不直接暴露到公网；
4. 生产环境只开放必要端口；
5. 对 Docker 容器配置 `restart: always`，提高服务可用性；
6. 定期清理无用镜像和日志文件；
7. 对核心服务保留日志，便于故障追踪；
8. 对部署脚本和 Compose 文件进行版本管理；
9. 部署前先在测试环境验证；
10. 更新 Nginx 配置后必须执行 `nginx -t` 检查语法。

## 附录：常用命令


本节整理后端部署、Docker Compose 管理、容器调试、端口排查和 Nginx 配置检查过程中常用的命令，便于部署和维护人员快速定位问题。

### Docker Compose 常用命令

进入项目部署目录：

```bash
cd /home/ecs-user/Intelligent_Curtain_Wall
```

启动全部服务：

```
docker compose up -d
```

停止全部服务：

```
docker compose down
```

重启全部服务：

```
docker compose restart
```

拉取最新镜像：

```
docker compose pull
```

拉取镜像并重新部署：

```
docker compose pull
docker compose up -d
```

查看 Compose 服务状态：

```
docker compose ps
```

查看全部服务日志：

```
docker compose logs
```

实时查看全部服务日志：

```
docker compose logs -f
```

查看指定服务日志：

```
docker compose logs -f user-authentication
```

重新创建指定服务：

```
docker compose up -d --force-recreate user-authentication
```

停止指定服务：

```
docker compose stop user-authentication
```

启动指定服务：

```
docker compose start user-authentication
```

重启指定服务：

```
docker compose restart user-authentication
```

------

### Docker 容器常用命令

查看正在运行的容器：

```
docker ps
```

查看所有容器，包括已停止容器：

```
docker ps -a
```

查看容器日志：

```
docker logs user-authentication
```

实时查看容器日志：

```
docker logs -f user-authentication
```

查看最近 100 行日志：

```
docker logs --tail=100 user-authentication
```

进入容器内部：

```
docker exec -it user-authentication bash
```

如果容器内没有 bash，则使用：

```
docker exec -it user-authentication sh
```

查看容器详细信息：

```
docker inspect user-authentication
```

查看容器资源占用：

```
docker stats
```

停止容器：

```
docker stop user-authentication
```

启动容器：

```
docker start user-authentication
```

重启容器：

```
docker restart user-authentication
```

删除已停止容器：

```
docker rm user-authentication
```

------

### Docker 镜像常用命令

查看本地镜像：

```
docker images
```

拉取指定镜像：

```
docker pull clearpool/intelligent-curtain-wall:user-authentication-new
```

删除指定镜像：

```
docker rmi clearpool/intelligent-curtain-wall:user-authentication-new
```

清理悬空镜像：

```
docker image prune -f
```

清理未使用的镜像、容器和网络：

```
docker system prune -f
```

查看 Docker 磁盘占用：

```
docker system df
```

------

### 端口与网络排查命令

查看所有监听端口：

```
sudo ss -tulpn
```

查看指定端口占用情况：

```
sudo ss -tulpn | grep 8008
```

查看 18001 端口是否监听：

```
sudo ss -tulpn | grep 18001
```

测试本机后端服务是否可访问：

```
curl http://127.0.0.1:8008
```

测试指定接口：

```
curl http://127.0.0.1:8008/api/account/login
```

测试外部访问：

```
curl http://服务器公网IP:8008
```

测试容器内部服务：

```
docker exec -it user-authentication sh
curl http://127.0.0.1:8000
```

查看 Docker 网络：

```
docker network ls
```

查看指定 Docker 网络详情：

```
docker network inspect intelligent_curtain_wall_curtainwall-net
```

------

### Nginx 常用命令

检查 Nginx 配置语法：

```
sudo nginx -t
```

重新加载 Nginx 配置：

```
sudo systemctl reload nginx
```

重启 Nginx：

```
sudo systemctl restart nginx
```

查看 Nginx 运行状态：

```
sudo systemctl status nginx
```

查看 Nginx 错误日志：

```
sudo tail -f /var/log/nginx/error.log
```

查看 Nginx 访问日志：

```
sudo tail -f /var/log/nginx/access.log
```

查看 Nginx 配置文件：

```
sudo vim /etc/nginx/nginx.conf
```

查看站点配置文件：

```
sudo vim /etc/nginx/sites-available/default
```

------

### 系统服务与防火墙命令

查看 Docker 服务状态：

```
sudo systemctl status docker
```

重启 Docker 服务：

```
sudo systemctl restart docker
```

设置 Docker 开机自启：

```
sudo systemctl enable docker
```

查看防火墙状态：

```
sudo ufw status
```

开放指定端口：

```
sudo ufw allow 8008/tcp
```

开放多个后端端口示例：

```
sudo ufw allow 18001/tcp
sudo ufw allow 18005/tcp
sudo ufw allow 18007/tcp
sudo ufw allow 18009/tcp
```

重新加载防火墙：

```
sudo ufw reload
```

------

### 常用健康检查命令

检查用户认证服务：

```
curl http://127.0.0.1:8008
```

检查裂缝检测服务：

```
curl http://127.0.0.1:18001
```

检查韧性评估服务：

```
curl http://127.0.0.1:18005
```

检查污渍检测服务：

```
curl http://127.0.0.1:18007
```

检查振动检测服务：

```
curl http://127.0.0.1:18009
```

------

### 部署脚本执行命令

进入脚本目录：

```
cd /home/ecs-user/Intelligent_Curtain_Wall/scripts
```

赋予脚本执行权限：

```
chmod +x deploy.sh
```

执行部署脚本：

```
./deploy.sh
```

后台执行部署脚本并保存日志：

```
nohup ./deploy.sh > deploy.log 2>&1 &
```

查看部署日志：

```
tail -f deploy.log
```

------

### 常见故障快速定位命令

容器是否运行：

```
docker ps -a
```

服务是否监听端口：

```
sudo ss -tulpn | grep 端口号
```

容器日志是否报错：

```
docker logs --tail=100 容器名称
```

Nginx 配置是否正确：

```
sudo nginx -t
```

Nginx 是否正常运行：

```
sudo systemctl status nginx
```

服务器磁盘是否占满：

```
df -h
```

查看内存占用：

```
free -h
```

查看 CPU 和进程情况：

```
top
```

查看系统日志：

```
journalctl -xe
```

查看 Docker 服务日志：

```
journalctl -u docker -f
```
