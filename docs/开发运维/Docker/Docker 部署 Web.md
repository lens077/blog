## 编写 Dockerfile

```conf
# /etc/nginx/conf.d/default.conf
server {
    listen 3000;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html index.htm;

    # Gzip 压缩配置
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied expired no-cache no-store private auth;
    gzip_types text/plain text/css text/xml text/javascript application/javascript application/xml+rss application/json;

    # 安全头设置
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;

    # 静态资源缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # 处理 SPA 路由 - 所有非静态文件请求都返回 index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 健康检查端点
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # 禁止访问隐藏文件
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    # 错误页面配置
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}

# 可选的 HTTP 重定向到 HTTPS (如果需要)
# server {
#     listen 80;
#     server_name your-domain.com;
#     return 301 https://$server_name$request_uri;
# }

```
1. 最简单方式是在项目根目录编写`Dockerfile`

`Dockerfile`
重要参数说明：
1. `RUN corepack enable && corepack prepare pnpm@latest --activate`
**含义**：启用 Node.js 内置的 **Corepack** 工具，并准备最新版本的 pnpm。

- **`corepack enable`**  
    Corepack 是 Node.js 官方提供的**包管理器版本管理工具**（从 Node.js 16.9.0 开始内置）。运行此命令后，Node.js 会在 `PATH` 中创建 `pnpm`、`yarn` 等命令的符号链接，指向 Corepack 管理的版本。
    
- **`corepack prepare pnpm@latest --activate`**  
    下载并缓存指定版本的 pnpm（此处为 `latest`），并将其激活为当前环境使用的 pnpm 版本。

为什么需要它？
- **保证环境一致性**：不同环境下可能安装了不同版本的 pnpm（或根本没有 pnpm）。使用 Corepack 可以确保 Docker 容器内使用的 pnpm 版本与项目锁文件 (`pnpm-lock.yaml`) 期望的版本一致。
- **避免 `pnpm: command not found`**：如果基础镜像 `node:22-alpine` 没有预装 pnpm，就需要用此方式安装。
2. `RUN pnpm install --frozen-lockfile`

**含义**：严格按照 `pnpm-lock.yaml` 中记录的版本和依赖树安装依赖，**不更新锁文件**。
`--frozen-lockfile` 的作用：
- **禁止修改 `pnpm-lock.yaml`**：如果安装过程中发现需要更新锁文件（例如依赖版本与 `package.json` 中的范围不匹配），命令会直接报错退出，而不是静默更新锁文件。
- **保证构建可重现**：确保 Docker 构建时安装的依赖与本地开发时完全一致，避免因依赖版本漂移导致难以排查的构建失败。

```Dockerfile
FROM node:16-alpine AS builder

WORKDIR /web
COPY . /web

# 分离环境
RUN cd /web
# 添加国内源加速下载
RUN npm config set registry https://registry.npmmirror.com
# 本项目使用 pnpm 管理包, 根据你的实际来修改
RUN npm install pnpm -g 
# 验证 pnpm安装与否
RUN pnpm -v
# 删除 npm 下载的包,由 pnpm 接管
RUN rm -f node_modules
# 安装依赖
ENV CI=true
ENV NODE_ENV=production
RUN corepack enable && corepack prepare pnpm@latest --activate
RUN pnpm install --frozen-lockfile
RUN build
# RUN pnpm install --no-frozen-lockfile 不根据package.lock的包来下载

# 处理下载出错
RUN RET=$? if [[ RET -ne 0 ]]; then tail -n 100 $HOME/.npm/_logs/*.log; exit $RST; fi;

# 打包
RUN pnpm build

# 使用nginx代理 web 项目的映射
FROM nginx:latest

# 把上一步骤打包好的的 dist目录传递到 nginx 默认的 html 目录作为映射
COPY --from=builder /web/dist/ /usr/share/nginx/html/

# 挂载 Nginx 的必要文件, 
# /etc/nginx/ssl/是 SSL 配置,不需要删掉即可
# /var/log/nginx/ 是 nginx 日志,不需要删掉即可
# /var/cache/nginx/是 nginx 缓存, 不需要删掉即可
# /etc/nginx/conf/ 是 nginx 的配置文件, 必须保留
# /usr/share/nginx/html/是 nginx 的 html 目录, 必须保留
VOLUME ["/var/cache/nginx/", "/etc/nginx/ssl/", "/etc/nginx/conf/", "/etc/nginx/conf/nginx.conf", "/var/log/nginx/", "/usr/share/nginx/html/"]

# 映射的端口
EXPOSE 80
EXPOSE 443

# 运行 nginx 服务
CMD ["nginx", "-g", "daemon off;"]

```

2. 构建 Docker 镜像

> 将`image`: `web`  # 这里需要替换成你使用 `Docker build`的名字

语法:

```shell
docker build [--progress=plain] [--no-cache] -t <image_name> <path>
```

例: 在当前目录构建所有文件, 镜像名为`web`, 不使用`Docker`缓存, 显示构建过程的详细信息

- --progress=plain: 构建过程中显示的详细信息的格式
- --no-cache : 不使用缓存
- web : 构建的镜像名
- . : 当前目录所有文件

```shell
docker build --progress=plain --no-cache -t web .
```

> 如果不在当前目录, 使用选项`-f`

例:

```shell
docker-compose -f ./build/ci/frontent/docker-compose.yml up -d
```

## 编写 Nginx 配置文件

创建 `nginx` 的目录, `html`与`conf`目录是必要的, 其他为可选

```shell
mkdir -p /home/nginx/html
mkdir -p /home/nginx/conf
mkdir -p /home/nginx/ssl
mkdir -p /home/nginx/log
```

## 编写docker-compose

1. 创建`docker-compose.yml`文件:

`docker-compose.yml`

```yml
services:
  nginx:
    image: web  # 这里需要替换成你的镜像名
    ports:
      - "80:80"
      - "443:443"
    restart: unless-stopped
    volumes:
      - /home/nginx/cache:/var/cache/nginx
      - /home/nginx/ssl:/etc/nginx/ssl
      - /home/nginx/conf:/etc/nginx/conf
      - /home/nginx/log:/var/log/nginx
    container_name: nginx

```

2. 部署:

```shell
docker-compose up -d 
```

## 选项

一些选项参考:

- down: 停止运行

```shell
docker-compose -f ./build/ci/frontent/docker-compose.yml down
```

- --build: `--build`：这是 `docker-compose up` 命令的另一个选项，它表示在启动容器之前需要构建镜像。如果在之前的构建中没有更改过镜像，
  `docker-compose` 将重用现有的镜像，但使用 `--build` 选项可以强制进行镜像构建，以确保使用最新的代码和配置。

```shell
docker-compose up -d --build
```
