# Docker 部署指引

[![][docker-release-shield]][docker-release-link]
[![][docker-size-shield]][docker-size-link]
[![][docker-pulls-shield]][docker-pulls-link]

我们提供了 [Docker 镜像][docker-release-link]，供你在自己的私有设备上部署 LobeChat 服务

#### TOC

- [安装 Docker 容器环境](#安装-docker-容器环境)
- [部署容器镜像](#部署容器镜像)
  - [`A` 指令部署（推荐）](#a-指令部署推荐)
  - [`B` Docker Compose](#b-docker-compose)

## 安装 Docker 容器环境

如果已安装，请跳过此步

**Ubuntu**

```fish
$ apt install docker.io
```

**CentOS**

```fish
$ yum install docker
```

## 部署容器镜像

### `A` 指令部署（推荐）

使用以下命令即可使用一键启动 LobeChat 服务：

```fish
$ docker run -d -p 3210:3210 \
  -e OPENAI_API_KEY=sk-xxxx \
  -e ACCESS_CODE=lobe66 \
  --name lobe-chat \
  lobehub/lobe-chat
```

> \[!NOTE]
>
> - 默认映射端口为 `3210`, 请确保未被占用或手动更改端口映射
> - 使用你的 OpenAI API Key 替换上述命令中的 `sk-xxxx`
> - 官方 Docker 镜像中设定的密码默认为 `lobe66`，请将其替换为自己的密码以提升安全性
> - LobeChat 支持的完整环境变量列表请参考 [环境变量](https://github.com/lobehub/lobe-chat/wiki/Environment-Variable.zh-CN) 部分

> \[!WARNING]
>
> 注意，当**部署架构与镜像的不一致时**，需要对 **Sharp** 进行交叉编译，详见 [Sharp 交叉编译](https://sharp.pixelplumbing.com/install#cross-platform)

#### 使用代理地址

如果你需要通过代理使用 OpenAI 服务，你可以使用 `OPENAI_PROXY_URL` 环境变量来配置代理地址：

```fish
$ docker run -d -p 3210:3210 \
  -e OPENAI_API_KEY=sk-xxxx \
  -e OPENAI_PROXY_URL=https://api-proxy.com/v1 \
  -e ACCESS_CODE=lobe66 \
  --name lobe-chat \
  lobehub/lobe-chat
```

> \[!NOTE]
>
> 由于官方的 Docker 镜像构建大约需要半小时左右，如果在更新部署后会出现「存在更新」的提示，可以等待镜像构建完成后再次部署。

#### Crontab 自动更新脚本

如果你想自动获得最新的镜像，你可以如下操作。

首先，新建一个 `lobe.env` 配置文件，内容为各种环境变量，例如：

```env
OPENAI_API_KEY=sk-xxxx
OPENAI_PROXY_URL=https://api-proxy.com/v1
ACCESS_CODE=arthals2333
CUSTOM_MODELS=-gpt-4,-gpt-4-32k,-gpt-3.5-turbo-16k,gpt-3.5-turbo-1106=gpt-3.5-turbo-16k,gpt-4-0125-preview=gpt-4-turbo,gpt-4-vision-preview=gpt-4-vision
```

然后，你可以使用以下脚本来自动更新：

```bash
#!/bin/bash
# auto-update-lobe-chat.sh

# 设置代理（可选）
export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 all_proxy=socks5://127.0.0.1:7890

# 拉取最新的镜像并将输出存储在变量中
output=$(docker pull lobehub/lobe-chat:latest 2>&1)

# 检查拉取命令是否成功执行
if [ $? -ne 0 ]; then
  exit 1
fi

# 检查输出中是否包含特定的字符串
echo "$output" | grep -q "Image is up to date for lobehub/lobe-chat:latest"

# 如果镜像已经是最新的，则不执行任何操作
if [ $? -eq 0 ]; then
  exit 0
fi

echo "Detected Lobe-Chat update"

# 删除旧的容器
echo "Removed: $(docker rm -f Lobe-Chat)"

# 运行新的容器
echo "Started: $(docker run -d --network=host --env-file /path/to/lobe.env --name=Lobe-Chat --restart=always lobehub/lobe-chat)"

# 打印更新的时间和版本
echo "Update time: $(date)"
echo "Version: $(docker inspect lobehub/lobe-chat:latest | grep 'org.opencontainers.image.version' | awk -F'"' '{print $4}')"

# 清理不再使用的镜像
docker images | grep 'lobehub/lobe-chat' | grep -v 'latest' | awk '{print $3}' | xargs -r docker rmi > /dev/null 2>&1
echo "Removed old images."
```

此脚本可以在 Crontab 中使用，但请确认你的 Crontab 可以找到正确的 Docker 命令。建议使用绝对路径。

配置 Crontab，每 5 分钟执行一次脚本：

```bash
*/5 * * * * /path/to/auto-update-lobe-chat.sh >> /path/to/auto-update-lobe-chat.log 2>&1
```

### `B` Docker Compose

使用 `docker-compose` 时配置文件如下:

```yml
version: '3.8'

services:
  lobe-chat:
    image: lobehub/lobe-chat
    container_name: lobe-chat
    restart: always
    ports:
      - '3210:3210'
    environment:
      OPENAI_API_KEY: sk-xxxx
      OPENAI_PROXY_URL: https://api-proxy.com/v1
      ACCESS_CODE: lobe66
```

#### Crontab 自动更新脚本

类似地，你可以使用以下脚本来自动更新 Lobe Chat，使用 `Docker Compose` 时，环境变量无需额外配置。

```bash
#!/bin/bash
# auto-update-lobe-chat.sh

# 设置代理（可选）
export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 all_proxy=socks5://127.0.0.1:7890

# 拉取最新的镜像并将输出存储在变量中
output=$(docker pull lobehub/lobe-chat:latest 2>&1)

# 检查拉取命令是否成功执行
if [ $? -ne 0 ]; then
  exit 1
fi

# 检查输出中是否包含特定的字符串
echo "$output" | grep -q "Image is up to date for lobehub/lobe-chat:latest"

# 如果镜像已经是最新的，则不执行任何操作
if [ $? -eq 0 ]; then
  exit 0
fi

echo "Detected Lobe-Chat update"

# 删除旧的容器
echo "Removed: $(docker rm -f Lobe-Chat)"

# 也许需要先进入 `docker-compose.yml` 所在的目录
# cd /path/to/docker-compose-folder

# 运行新的容器
echo "Started: $(docker-compose up)"

# 打印更新的时间和版本
echo "Update time: $(date)"
echo "Version: $(docker inspect lobehub/lobe-chat:latest | grep 'org.opencontainers.image.version' | awk -F'"' '{print $4}')"

# 清理不再使用的镜像
docker images | grep 'lobehub/lobe-chat' | grep -v 'latest' | awk '{print $3}' | xargs -r docker rmi > /dev/null 2>&1
echo "Removed old images."
```

此脚本亦可以在 Crontab 中使用，但请确认你的 Crontab 可以找到正确的 Docker 命令。建议使用绝对路径。

配置 Crontab，每 5 分钟执行一次脚本：

```bash
*/5 * * * * /path/to/auto-update-lobe-chat.sh >> /path/to/auto-update-lobe-chat.log 2>&1
```

[docker-pulls-link]: https://hub.docker.com/r/lobehub/lobe-chat
[docker-pulls-shield]: https://img.shields.io/docker/pulls/lobehub/lobe-chat?color=45cc11&labelColor=black&style=flat-square
[docker-release-link]: https://hub.docker.com/r/lobehub/lobe-chat
[docker-release-shield]: https://img.shields.io/docker/v/lobehub/lobe-chat?color=369eff&label=docker&labelColor=black&logo=docker&logoColor=white&style=flat-square
[docker-size-link]: https://hub.docker.com/r/lobehub/lobe-chat
[docker-size-shield]: https://img.shields.io/docker/image-size/lobehub/lobe-chat?color=369eff&labelColor=black&style=flat-square
