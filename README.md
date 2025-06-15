# n8n

本项目提供一个 `docker-compose.yml` 文件，用于部署一个增强版的 [n8n](https://n8n.io/) 实例。

与官方的 Docker 镜像相比，此配置的核心优势在于：**在容器启动时自动安装并配置一个持久化的 Python 3 虚拟环境**。这使得用户可以在 n8n 的 `Code` 节点中无缝地执行 Python 脚本，并且安装的库会在容器重启后依然存在。

## 与官方镜像的主要区别和修改点

为了实现持久化的 Python 环境，我们对官方 `docker.n8n.io/n8nio/n8n` 镜像的启动流程进行了一些关键修改：

1.  **提升初始权限 (`user: root`)**

      * **目的**: 默认情况下，n8n 容器以低权限的 `node` 用户运行，无法安装系统级软件包。我们临时将用户切换到 `root`，以便有权限执行 `apk add` 命令来安装 `py3-virtualenv`。

2.  **自定义启动命令 (`command`)**

      * **目的**: 注入我们自己的初始化脚本，在 n8n 主程序启动前完成环境配置。
      * **执行流程**:
        1.  `apk add --no-cache py3-virtualenv`: 安装 Python 虚拟环境管理工具。
        2.  `python3 -m venv /data/venv`: 在 `/data` 目录下创建一个名为 `venv` 的 Python 虚拟环境。
        3.  `chown -R node:node /data/venv`: 将新建的 `venv` 目录的所有权交还给 `node` 用户。这是**至关重要**的一步，确保 n8n 进程（以 `node` 用户身份运行）有权限读写该目录，从而可以管理 Python 库。
        4.  `exec su -l node -c '/docker-entrypoint.sh \"$@\"'`: 在完成所有 `root` 权限的操作后，**切换回 `node` 用户**，并执行官方的原始启动脚本。这确保了 n8n 应用本身在安全的环境中运行，与官方设置保持一致。

3.  **新增持久化卷 (`data_venv`)**

      * **目的**: 将 Python 虚拟环境 (`/data/venv`) 保存到 Docker-managed volume `data_venv` 中。
      * **优势**: 当你重新启动或更新 n8n 容器时，已经创建的虚拟环境和通过 `pip` 安装的所有 Python 库都会被保留，无需每次都重新安装。

4.  **启用代码执行环境变量**

      * `NODE_FUNCTION_ALLOW_BUILTIN=*` 和 `NODE_FUNCTION_ALLOW_EXTERNAL=*` 这两个环境变量被设置，以允许 n8n 的 `Code` 节点执行外部命令，这是我们能够调用 Python 环境的前提。

## 部署指南

### 前提条件

在开始之前，请确保你的系统上已经安装了 [Docker](https://www.docker.com/get-started) 和 [Docker Compose](https://docs.docker.com/compose/install/)。

### 文件结构

将以下内容保存为 `docker-compose.yml` 文件。

```yaml
# docker-compose.yml
version: '3.8'
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5679:5678" # 官方默认的是 5678:5678
    volumes:
      - data:/home/node/.n8n
      - data_venv:/data # 虚拟环境将保存在这里
    environment:
      - NODE_FUNCTION_ALLOW_BUILTIN=*
      - NODE_FUNCTION_ALLOW_EXTERNAL=*
      - GENERIC_TIMEZONE=Asia/Shanghai
      - TZ=Asia/Shanghai
    user: root
    entrypoint: ["tini", "--"]
    command: >
      /bin/sh -c "
      apk add --no-cache py3-virtualenv && \
      python3 -m venv /data/venv && \
      chown -R node:node /data/venv && \
      chmod +x /docker-entrypoint.sh && \
      exec su -l node -c '/docker-entrypoint.sh \"$@\"'
      "
    networks:
      - default

volumes:
  data:
  data_venv: # 为虚拟环境定义 volume

networks:
  default:
    external: true
    name: n8n # 可以自定义网络名，或加入已有的网络
```

### 部署步骤 (Linux / macOS / Windows)

以下命令需要在终端 (Linux/macOS) 或 PowerShell/CMD (Windows) 中执行。

**第一步：下载 docker-compose**

手动下载上面的 `docker-compose.yml` 放在任意目录中，或者使用以下命令下载到终端当前目录

```sh
wget https://raw.githubusercontent.com/osen77/n8n/master/docker-compose.yml
```

**第二步：启动 n8n 容器**

在 `docker-compose.yml` 文件所在的目录中，运行以下命令来启动 n8n 服务：

```sh
docker-compose up -d
```

`-d` 参数会让容器在后台运行。容器首次启动时，会自动完成 Python 环境的安装和配置。

**第三步：访问 n8n**

打开你的浏览器，访问 `http://localhost:5679`。

**如何停止服务**

如果需要停止 n8n 服务，运行以下命令：

```sh
docker-compose down
```

此命令会停止并移除容器，但通过 `volumes` 定义的 `data` 和 `data_venv` 卷中的数据（包括你的 n8n 工作流和 Python 库）将会被保留。

## 如何在 n8n 中使用 Python

部署完成后，你可以在 n8n 的 `execute command` 节点中使用这个持久化的 Python 环境。

1.  Python 可执行文件路径:
    使用以下路径来调用 Python 解释器：

    ```
    /data/venv/bin/python3
    ```
2.  在 execute command 节点中安装第三方包：
    在 n8n 工作流中添加一个 execute command 节点。
    在该节点中运行命令来安装第三方包。例如，安装 pandas 的命令是 /data/venv/bin/pip3 install pandas。这里的 /data/venv/bin/pip3 指向你之前创建并持久化的虚拟环境中的 pip 命令。

3.  在 execute command 节点中执行外部 Python 文件：
    将你的 Python 文件（例如 xxx.py）放在你储存卷的目录中（例如 /home/node/.n8n/ 或其子文件夹）。
    在 execute command 节点中，使用虚拟环境中的 Python 解释器来执行你的 Python 文件。例如：/data/venv/bin/python3 /home/node/.n8n/xxx.py
