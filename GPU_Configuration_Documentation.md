# 矩池云 GPU 环境配置文档

> [!TIP]
> 服务器环境已保存为 ICW-Matpool-Environment

## 租用配置与硬件信息

![](assets/2024-11-25_03-30-29.png)

![](assets/2024-11-24_14-32-08.png)

## 配置 Git LFS

安装 Git LFS：

```bash
sudo apt-get install git-lfs
```

初始化 Git LFS：

```bash
git lfs install
```

## 配置 Anaconda

在用户目录下载并安装 Anaconda：

```bash
cd ~
wget https://repo.anaconda.com/archive/Anaconda3-2024.10-1-Linux-x86_64.sh
chmod +x Anaconda3-2024.10-1-Linux-x86_64.sh
./Anaconda3-2024.10-1-Linux-x86_64.sh
rm Anaconda3-2024.10-1-Linux-x86_64.sh
```

验证安装成功：

```bash
conda --version
```

![](assets/2024-11-24_15-53-12.png)

配置 Anaconda 使用清华镜像源：

```bash
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2/
conda config --set show_channel_urls yes
```

配置 pip 使用清华镜像源：

```bash
cd ~
mkdir .pip
cd .pip/
touch pip.conf
vim pip.conf
```

将 `pip.conf` 的内容修改为：

```
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
```

输入 `:wq` 保存。

## 配置 Docker Compose

更新系统软件包列表并安装 Docker Compose：

```bash
sudo apt update
sudo apt install docker-compose
```

验证安装成功：

```bash
docker --version
docker-compose --version
```

![](assets/2024-11-24_15-59-01.png)

## 配置 Trojan

更新系统软件包列表并安装 Trojan：

```bash
sudo apt update
sudo apt install trojan
```

验证安装成功：

```bash
trojan --version
```

![](assets/2024-11-24_16-01-26.png)

编辑 Trojan 配置文件：

```bash
sudo nano /etc/trojan/config.json
```

```json
{
    "run_type": "client",
    "local_addr": "127.0.0.1",
    "local_port": 1080,
    "remote_addr": "139.159.217.107",
    "remote_port": 4038,
    "password": [
        "4806276b-11a2-38da-9a2c-bc6524ab0cb6"
    ],
    "log_level": 1,
    "ssl": {
        "sni": "baidu.com",
        "verify": false,
        "cipher": "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384",
        "cipher_tls13": "TLS_AES_128_GCM_SHA256:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_256_GCM_SHA384",
        "prefer_server_cipher": true,
        "alpn": [
            "http/1.1"
        ],
        "reuse_session": true,
        "session_ticket": false,
        "session_timeout": 600
    },
    "tcp": {
        "prefer_ipv4": false,
        "no_delay": true,
        "keep_alive": true,
        "reuse_port": false,
        "fast_open": false,
        "fast_open_qlen": 20
    }
}
```

启动 Trojan 服务：

```bash
sudo systemctl start trojan
sudo systemctl enable trojan
```

查看 Trojan 服务端口占用情况：

```bash
sudo netstat -tulpn | grep 1080
```

![](assets/2024-11-24_16-23-58.png)

## 配置 Docker 代理

编辑 Docker 代理配置文件：

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo nano /etc/systemd/system/docker.service.d/http-proxy.conf
```

```
[Service]
Environment="HTTP_PROXY=socks5://127.0.0.1:1080"
Environment="HTTPS_PROXY=socks5://127.0.0.1:1080"
Environment="NO_PROXY=localhost,127.0.0.1"
```

重启 Docker 服务：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

查看 Docker 环境变量：

```bash
sudo systemctl show --property=Environment docker
```

![](assets/2024-11-24_17-07-49.png)

## 配置 NVIDIA Docker 支持

编辑 Docker Daemon 配置文件：

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```

重启 Docker 服务：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

检查 NVIDIA 运行时是否配置成功：

```bash
sudo docker info | grep Runtimes
```

![](assets/2024-11-25_02-31-27.png)

## 配置 Docker Compose 部署任务

编辑 `docker-compose.yml` 文件：

```bash
sudo nano /home/mat/Intelligent_Curtain_Wall/docker-compose.yml
```

```yml
services:
  # 金属幕墙锈蚀污损检测系统
  corrosion-detection:
    image: minmuslin/intelligent-curtain-wall:corrosion-detection
    container_name: corrosion-detection
    ports:
      - "8000:8080"
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all

  # sudo docker run -d --name corrosion-detection -p 8000:8080 --runtime=nvidia -e NVIDIA_VISIBLE_DEVICES=all minmuslin/intelligent-curtain-wall:corrosion-detection

  # 石材幕墙裂缝检测系统
  crack-detection:
    image: minmuslin/intelligent-curtain-wall:crack-detection
    container_name: crack-detection
    ports:
      - "8001:8080"
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    volumes:
      - /home/mat/Intelligent_Curtain_Wall/crack-detection:/segformerProject/model

  # sudo docker run -d --name crack-detection -p 8001:8080 --runtime=nvidia -e NVIDIA_VISIBLE_DEVICES=all -v /home/mat/Intelligent_Curtain_Wall/crack-detection:/segformerProject/model minmuslin/intelligent-curtain-wall:crack-detection

  # 玻璃幕墙平整度检测系统
  flatness-detection:
    image: minmuslin/intelligent-curtain-wall:flatness-detection
    container_name: flatness-detection
    ports:
      - "8002:8080"
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    volumes:
      - /home/mat/Intelligent_Curtain_Wall/flatness-detection:/app/model

  # sudo docker run -d --name flatness-detection -p 8002:8080 --runtime=nvidia -e NVIDIA_VISIBLE_DEVICES=all -v /home/mat/Intelligent_Curtain_Wall/flatness-detection:/app/model minmuslin/intelligent-curtain-wall:flatness-detection

  # 移动端幕墙数据采集与展示系统
  mobile-data:
    image: minmuslin/intelligent-curtain-wall:mobile-data
    container_name: mobile-data
    ports:
      - "8003:8080"
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all

  # sudo docker run -d --name mobile-data -p 8003:8080 --runtime=nvidia -e NVIDIA_VISIBLE_DEVICES=all minmuslin/intelligent-curtain-wall:mobile-data

  # 移动端幕墙数据采集与展示系统（数据记录程序）
  mobile-data-data-logger:
    image: minmuslin/intelligent-curtain-wall:mobile-data-data-logger
    container_name: mobile-data-data-logger
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all

  # sudo docker run -d --name mobile-data-data-logger --runtime=nvidia -e NVIDIA_VISIBLE_DEVICES=all minmuslin/intelligent-curtain-wall:mobile-data-data-logger

  # 无人机采集数据的 3D 建模与通讯系统
  modeling-communication:
    image: minmuslin/intelligent-curtain-wall:modeling-communication
    container_name: modeling-communication
    ports:
      - "8004:8080"
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all

  # sudo docker run -d --name modeling-communication -p 8004:8080 --runtime=nvidia -e NVIDIA_VISIBLE_DEVICES=all minmuslin/intelligent-curtain-wall:modeling-communication

  # 幕墙韧性评估软件系统
  resilience-assessment:
    image: minmuslin/intelligent-curtain-wall:resilience-assessment
    container_name: resilience-assessment
    ports:
      - "8005:8080"
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all

  # sudo docker run -d --name resilience-assessment -p 8005:8080 --runtime=nvidia -e NVIDIA_VISIBLE_DEVICES=all minmuslin/intelligent-curtain-wall:resilience-assessment

  # 玻璃幕墙爆裂和平整度检测系统
  spalling-detection:
    image: minmuslin/intelligent-curtain-wall:spalling-detection
    container_name: spalling-detection
    ports:
      - "8006:8080"
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all

  # sudo docker run -d --name spalling-detection -p 8006:8080 --runtime=nvidia -e NVIDIA_VISIBLE_DEVICES=all minmuslin/intelligent-curtain-wall:spalling-detection

  # 石材幕墙污渍检测系统
  stain-detection:
    image: minmuslin/intelligent-curtain-wall:stain-detection
    container_name: stain-detection
    ports:
      - "8007:8080"
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all

  # sudo docker run -d --name stain-detection -p 8007:8080 --runtime=nvidia -e NVIDIA_VISIBLE_DEVICES=all minmuslin/intelligent-curtain-wall:stain-detection

  # 用户鉴权系统
  user-authentication:
    image: minmuslin/intelligent-curtain-wall:user-authentication
    container_name: user-authentication
    ports:
      - "8008:8080"
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all

  # sudo docker run -d --name user-authentication -p 8008:8080 --runtime=nvidia -e NVIDIA_VISIBLE_DEVICES=all minmuslin/intelligent-curtain-wall:user-authentication

  # 幕墙震动数据检测与展示系统
  vibration-detection:
    image: minmuslin/intelligent-curtain-wall:vibration-detection
    container_name: vibration-detection
    ports:
      - "8009:8080"
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all

  # sudo docker run -d --name vibration-detection -p 8009:8080 --runtime=nvidia -e NVIDIA_VISIBLE_DEVICES=all minmuslin/intelligent-curtain-wall:vibration-detection
```

编辑 `automated-deployment.sh` 文件：

```bash
sudo nano /home/mat/Intelligent_Curtain_Wall/automated-deployment.sh
```

```shell
#!/bin/bash

LOG_DIR="/home/mat/Intelligent_Curtain_Wall/deployment-logs"

export TZ="Asia/Shanghai"

TIMESTAMP=$(date +"%Y%m%d-%H:%M")
START_TIME=$(date +"%Y-%m-%d %H:%M:%S")
START_SECONDS=$(date +%s)

mkdir -p "$LOG_DIR"

{
    echo "Execution Start Time: $START_TIME"

    cd /home/mat/Intelligent_Curtain_Wall

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

## 配置 `mat` 用户权限

允许 `mat` 用户执行所有命令，且不需要输入密码：

```bash
sudo visudo
```

在最后一行添加如下配置：

```
mat ALL=(ALL) NOPASSWD: ALL
```

## 配置 SSH 密钥

生成 SSH 密钥对：

```bash
ssh-keygen -t rsa -b 4096 -C "minmuslin@outlook.com"
```

将公钥添加到 `~/.ssh/authorized_keys`：

```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

确保权限正确：

```bash
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

复制私钥内容：

```bash
cat ~/.ssh/id_rsa
```
