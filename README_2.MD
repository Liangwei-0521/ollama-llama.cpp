# 本地WSL连接web open ui异常处理

### **异常情况**

在WSL中成功安装ollama，浏览器中输入

```
http://127.0.0.1:11434
```

**显示**

```
Ollama is running
```

在window的WSL中输入ollama-ls命令，显示已经成功安装外部模型

```
(base) liangwei@LAPTOP-RBVCU2OF:~$ ollama ls
NAME                     ID              SIZE      MODIFIED
qwen2.5:1.5b-instruct    65ec06548149    986 MB    4 days ago
```

**检查服务端口**

web open ui 也成功安装，且8080端口成功暴露

```
(base) liangwei@LAPTOP-RBVCU2OF:~$ docker ps
CONTAINER ID   IMAGE                                COMMAND                  CREATED          STATUS                    PORTS                                      NAMES
2c14873595f4   ghcr.io/open-webui/open-webui:main   "bash start.sh"          28 minutes ago   Up 28 minutes (healthy)   8080/tcp                                   charming_morse
```

**11434端口成功被监听**

```
sudo lsof -i :11434
```

```
COMMAND   PID     USER   FD   TYPE  DEVICE SIZE/OFF NODE NAME
ollama  37652 liangwei    3u  IPv6 1669567      0t0  TCP *:11434 (LISTEN)
```


http://127.0.0.1:11434访问时输出 ollama is running的内容，但是我在web openui中无法通过这个网址外部连接到模型（输出报错：WebUI could not connect to Ollama），可以确保的是

```
在window的WSL中输出：
(base) liangwei@LAPTOP-RBVCU2OF:~$ ollama ls
NAME                     ID              SIZE      MODIFIEDqwen2.5:1.5b-instruct    65ec06548149    986 MB    4 days ago
```

```
在window的shell命令中输出的是：
C:\Users\qnhl>ollama ls
NAME                            ID              SIZE    MODIFIED
shaw/dmeta-embedding-zh:latest  55960d8a3a42    408 MB  4 days ago
qwen2.5:1.5b-instruct           65ec06548149    986 MB  4 days ago
```

我的web openui是通过docker 安装的，在外部链接使用http://host.docker.internal:11434/ 也报错：WebUI could not connect to Ollama

### **解决方案**

`docker run --add-host=host.docker.internal:172.18.25.236 -p 8080:8080 ghcr.io/open-webui/open-webui:main` 在启动 Docker 容器时执行了几项操作。下面我会详细解析每个部分的含义：

### 1. `docker run`

这是启动 Docker 容器的基本命令。它可以执行容器中的指定镜像，并根据提供的选项和参数对容器进行配置。

### 2. `--add-host=host.docker.internal:172.18.25.236`

这个选项用来在容器的 `/etc/hosts` 文件中手动添加一个主机名和 IP 地址的映射。

* `--add-host`：指定手动添加一个主机名到 IP 地址的映射。
* `host.docker.internal`：这是你希望在容器内解析的主机名。通常，它在 Windows 和 macOS 上自动指向宿主机 IP，但在 WSL 2 或 Linux 环境下需要手动添加。
* `172.18.25.236`：这是 WSL 2 中你的主机 IP 地址（通过 `ip addr show eth0` 得到的）。这个 IP 是 WSL 在虚拟网络中的地址。

因此，这部分操作是在 Docker 容器内创建一条新的主机名解析记录，将 `host.docker.internal` 解析为 `172.18.25.236`，这样容器内部可以通过 `host.docker.internal` 来访问 WSL 主机的服务（如 Ollama 服务）。

### 3. `-p 8080:8080`

这是端口映射选项，用来将主机（WSL）上的端口与容器内的端口进行绑定。

* `8080:8080`：将主机上的 8080 端口映射到容器的 8080 端口。这样，外部访问 `http://localhost:8080`（在 WSL 中访问 `http://172.18.25.236:8080`）时，会被转发到 Docker 容器中的 8080 端口上运行的服务。

### 4. `ghcr.io/open-webui/open-webui:main`

这是 Docker 镜像的名称。

* `ghcr.io/open-webui/open-webui`：这是存放在 GitHub Container Registry (GHCR) 中的 Docker 镜像。`open-webui` 是你运行的应用。
* `:main`：这是你拉取的镜像的版本标签，表示你正在运行这个镜像的 `main` 版本。

### 总结

这条命令启动了 `ghcr.io/open-webui/open-webui:main` 镜像对应的容器，并执行了以下任务：

* 在容器中手动将 `host.docker.internal` 解析为你的 WSL 主机 IP 地址 `172.18.25.236`。
* 将主机的 8080 端口映射到容器的 8080 端口，允许你通过 `localhost:8080` 或 `172.18.25.236:8080` 访问容器中的服务。通过这个设置，容器可以访问 WSL 主机的 Ollama 服务，并将 WebUI 的访问端口暴露在 8080。
* http://172.18.25.236:11434  才是WSL宿主机的IP地址
