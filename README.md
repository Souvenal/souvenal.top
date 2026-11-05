# souvenal.top

最小个人网站：Nginx 静态主页 + Cloudflare Tunnel。当前不包含数据库、账户系统或动态 API。

## 开发环境与生产环境

代码只有一份，运行方式分为两套。**不要分别维护两份主页代码。**

| | 开发环境 | 生产环境 |
|---|---|---|
| 配置文件 | `compose.dev.yaml` | `compose.yaml` |
| 访问地址 | <http://localhost:18080> | <https://souvenal.top> |
| Cloudflare Tunnel | 不启动 | 启动 |
| Tunnel Token | 不需要 | 只从本机 `.env` 读取 |
| 网站文件 | 挂载 `site/`，修改后刷新即可 | 构建时复制进镜像，发布后不受本机修改影响 |
| 本机端口 | 只监听 `127.0.0.1:18080` | 不发布端口 |
| 容器/网络前缀 | `souvenal-top-dev` | `souvenal-top` |

生产访问链路：

```text
浏览器 → Cloudflare HTTPS → cloudflared 容器 → web 容器（Nginx）
```

## 一、日常开发

启动开发环境：

```powershell
docker-compose -f compose.dev.yaml up -d
```

打开 <http://localhost:18080>。修改 `site/index.html`、`site/styles.css` 或 `site/favicon.svg` 后，刷新浏览器即可，不需要重新构建镜像。

如果修改了 `nginx.conf`，需要重启开发容器：

```powershell
docker-compose -f compose.dev.yaml restart web
```

查看状态和日志：

```powershell
docker-compose -f compose.dev.yaml ps
docker-compose -f compose.dev.yaml logs --tail=100 web
```

停止开发环境：

```powershell
docker-compose -f compose.dev.yaml down
```

这条命令只处理开发环境，不会停止生产环境。

## 二、提交代码

推荐使用简短的功能分支，让 `main` 始终保持可以发布的状态：

1. 从 `main` 创建功能分支，例如 `feature/update-homepage`。
2. 在开发环境修改并验证。
3. 用 `git diff` 检查改动，确认没有 `.env` 或 Token。
4. 提交代码，并把功能分支合并回 `main`。
5. 从 `main` 构建和发布生产环境。

常用检查：

```powershell
git status
git diff
```

## 三、首次配置生产环境

按照 [Cloudflare 官方文档](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/create-remote-tunnel/)操作：

1. 确认 `souvenal.top` 已在 Cloudflare 中且状态为 Active。
2. 打开 **Cloudflare Dashboard → Networking → Tunnels → Create a tunnel**，名称可填 `souvenal-top`。
3. 安装环境选择 **Docker**，只复制命令中 `--token` 后面的长字符串。
4. 在项目目录创建本机环境变量文件：

   ```powershell
   Copy-Item .env.example .env
   ```

5. 打开 `.env`，把 Token 填到 `TUNNEL_TOKEN=` 后面。**不要提交、截图或发送这个文件。**
6. 在 Tunnel 的 **Routes → Published application** 中添加路由：
   - Subdomain：留空（主域名）
   - Domain：`souvenal.top`
   - Path：留空
   - Service URL：`http://web:80`
7. 保存路由。Cloudflare 通常会自动创建 Tunnel DNS 记录。如果已有同名网站 A/AAAA/CNAME 记录导致冲突，只处理冲突的网站记录，不要误删 MX 邮件记录。

不需要开放路由器端口，也不需要提供 Cloudflare 密码或 API Key。

## 四、发布生产环境

先完成开发验证和代码提交，再运行：

```powershell
docker-compose up -d --build
```

检查：

```powershell
docker-compose ps
docker-compose exec web wget -q -O - http://127.0.0.1/healthz
docker-compose logs --tail=100 cloudflared
```

最后访问 <https://souvenal.top>，同时检查电脑和手机页面。

生产环境没有本机预览端口；开发预览请使用 `compose.dev.yaml`。

## 五、修改后的标准发布流程

```text
功能分支修改
  → 开发环境验证
  → Git 提交并合并 main
  → 构建生产镜像
  → 更新生产容器
  → 检查真实域名
```

对应命令：

```powershell
# 1. 开发和验证
docker-compose -f compose.dev.yaml up -d

# 2. 验证、提交并合并代码后，发布
docker-compose up -d --build

# 3. 检查
docker-compose ps
docker-compose logs --tail=100 web
docker-compose logs --tail=100 cloudflared
```

如果发布失败，先不要反复修改生产环境。保留日志，让开发环境继续复现和修复；需要回滚时，恢复到上一个可用 Git 提交并重新构建发布。网站变复杂后，再增加镜像版本标签和一键回滚。

## 六、更新基础镜像

`nginx:alpine` 和 `cloudflare/cloudflared:latest` 是滚动标签，重启容器不会自动更新。

先在开发环境更新并验证：

```powershell
docker-compose -f compose.dev.yaml pull web
docker-compose -f compose.dev.yaml up -d --force-recreate
```

验证后再更新生产环境：

```powershell
docker-compose pull cloudflared
docker-compose build --pull web
docker-compose up -d
```

## 七、文件说明

- `site/index.html`：主页文案与结构。
- `site/styles.css`：主页样式。
- `site/favicon.svg`：浏览器图标。
- `nginx.conf`：开发和生产共用的 Nginx 配置。
- `compose.dev.yaml`：开发环境，只有本机 Nginx。
- `compose.yaml`：生产环境，独立网站镜像 + Cloudflare Tunnel。
- `Dockerfile`：生产网站镜像构建定义。
- `.env`：生产 Tunnel Token，已被 Git 和 Docker 构建上下文忽略。

## 八、运行注意事项

- 使用 Docker Compose 插件时，把文档中的 `docker-compose` 换成 `docker compose`。
- 如需开机自动运行生产环境，在 Docker Desktop 设置中启用登录时启动。
- 电脑关机或休眠期间，本地部署的生产网站会离线。
