# ARTEX Windows 本地构建与部署

> 官方 Docker 镜像（`autumn27/artex`）拉取失效时的替代路径：**数据库用 Docker 官方 postgres 镜像（官方源可直连），ARTEX 本体从源码编译成"内嵌前端"的单二进制本地运行**。全程不依赖任何第三方镜像。
>
> 实测环境（2026-10-12，全流程约 5 分钟）：Windows 11 + Git Bash + Go 1.27.2 + Node 24.16.0 + Docker Desktop。

## 一、前置环境

| 依赖 | 要求 | 说明 |
| --- | --- | --- |
| Go | **>= 1.26.3**（go.mod 声明） | 低版本会由 Go 工具链自动拉取；直接装最新即可 |
| Node.js + npm | >= 20 | 仅前端静态构建用（`node -v` 检查） |
| Docker Desktop | 任意版本 | **仅用来跑 PostgreSQL**，不涉及应用镜像 |
| Git Bash | 随 Git for Windows 附带 | 推荐用它执行本文档全部命令 |

命令统一在 **Git Bash** 里执行（避免 Windows cmd 下 npm 脚本 shell 的坑，见第 4 步）。

## 二、构建步骤（7 步）

### 1. 获取源码

```bash
git clone https://github.com/Autumn-27/ARTEX.git
cd ARTEX
```

### 2. 启动 PostgreSQL（唯一用到 Docker 的地方）

```bash
docker run -d --name artex-pg -p 5433:5432 \
  -e POSTGRES_USER=artex -e POSTGRES_PASSWORD=111111 -e POSTGRES_DB=artex \
  -v artex-pg:/var/lib/postgresql/data --restart unless-stopped postgres:16-alpine

# 等几秒后确认就绪（出现 accepting connections 即可）
docker exec artex-pg pg_isready -U artex -d artex
```

- 端口用 **5433** 而非 5432，避免与本机已装的 PostgreSQL 冲突（可自行改，但需与下一步 config.json 保持一致）。
- **已有可用 PostgreSQL 时跳过本步**，直接在下一步填它的连接信息。

### 3. 写数据库配置

在**项目根目录**创建 `config.json`（该文件已被 .gitignore，不影响源码）：

```json
{
  "database": {
    "host": "127.0.0.1",
    "port": 5433,
    "user": "artex",
    "password": "111111",
    "dbname": "artex",
    "sslmode": "disable"
  }
}
```

- **表结构不用手动建**：程序每次启动会幂等重跑 `db/schema.sql`（"重启即迁移"）。
- 也可用环境变量 `ARTEX_PG_DSN=postgres://artex:111111@127.0.0.1:5433/artex?sslmode=disable` 覆盖本文件。

### 4. 构建前端静态产物

```bash
cd web
npm ci          # registry 慢就先执行：npm config set registry https://registry.npmmirror.com
NEXT_EXPORT=1 npx next build
cd ..
```

产出目录：`web/out/`。

> ⚠️ **不要用 `npm run build:static`**：它的定义是 `NEXT_EXPORT=1 next build`（POSIX 变量前置语法），npm 在 Windows 下默认用 cmd.exe 执行脚本，会报 `'NEXT_EXPORT' 不是内部或外部命令`。用上面的 Git Bash + npx 写法；或先 `npm config set script-shell "C:\Program Files\Git\bin\bash.exe"` 再跑原脚本。

### 5. 把前端产物同步进内嵌目录

```bash
rm -rf server/webui/dist
cp -r web/out server/webui/dist
```

> ⚠️ **必须先删再拷**：目标目录已存在时，`cp -r web/out server/webui/dist` 会嵌套成 `server/webui/dist/out/`，二进制里多嵌一份近 20M 的重复文件（不影响运行，但体积翻倍）。

### 6. 编译单二进制

```bash
# Go 不在 PATH 时先补上（路径按实际安装位置改；标准安装在 C:\Program Files\Go\bin）
export PATH="/d/software/go/bin:$PATH"
# 国内网络加 Go 模块代理（能直连 proxy.golang.org 可跳过）
export GOPROXY=https://goproxy.cn,direct

CGO_ENABLED=0 go build -tags embedui -trimpath -o artex.exe ./cmd/artex
```

- **`-tags embedui` 必须有**：不加则前端不内嵌，启动后页面直接 404。
- 首次编译需联网拉依赖（`github.com/Autumn-27/norma` 等），约 1~3 分钟；成功后生成 `artex.exe`（约 50M）。
- CMD / PowerShell 等价写法：

  ```
  set PATH=D:\software\go\bin;%PATH%
  set GOPROXY=https://goproxy.cn,direct
  set CGO_ENABLED=0
  go build -tags embedui -trimpath -o artex.exe ./cmd/artex
  ```

### 7. 启动与验证

```bash
./start.bat        # 双击也可以；首次启动自动初始化数据库
```

```bash
# 另开终端验证
curl http://127.0.0.1:8787/api/health
# → {"ok":true,"service":"artex","version":"dev"}
```

- 默认监听 `:8787`（Web/API），流量代理 `127.0.0.1:8788`。
- **务必用 `start.bat` 启动，不要直接跑 `artex.exe`**：它是守护脚本，按退出码决定是否拉起（页面「一键更新 / 回滚」靠它完成换装）；直接跑本体的话更新完进程不会再起来。
- 数据落盘位置：`data/`（SQLite 等，exe 同级）、`jwt.key`（项目根，运行时生成）、PG 数据在 Docker 卷 `artex-pg`。

## 三、首次使用

1. 浏览器打开 **http://localhost:8787** —— 根路径会 307 跳转；未初始化时自动进 `/setup`。
2. 在 `/setup` 设置管理员密码（**登录用户名固定为 `ARTEX`**，不可改）。
3. LLM 在侧边栏「系统配置」里填 API Key（`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` 或自填 Base URL）；也可用环境变量启动。

## 四、常见问题排查

| 现象 | 原因 / 处置 |
| --- | --- |
| 首页 HTML 是 `__next_error__`、内容很少 | **正常**。根路由源码是 `redirect("/function/tasks")`，静态导出就是"错误页外壳 + 客户端跳转"，浏览器打开后自动跳转 |
| `go: command not found` | Go 未加入 PATH：用第 6 步的 `export PATH`，或全路径调用 |
| 编译长时间卡住 / 模块下载失败 | 加 `GOPROXY=https://goproxy.cn,direct`（第 6 步） |
| `'NEXT_EXPORT' 不是内部或外部命令` | npm 脚本 shell 坑，见第 4 步说明 |
| 编译报 `pattern all:webui/dist: no matching files` | 第 5 步没做：`server/webui/dist` 为空 |
| 启动报「未找到数据库配置」 | 项目根没有 `config.json`；或用 `ARTEX_CONFIG` / `ARTEX_PG_DSN` 环境变量指定 |
| 启动报数据库连接失败 | `docker ps` 看 `artex-pg` 是否 Up；核对端口 / 密码与 config.json 一致 |
| 忘记管理员密码 | `./reset-password.sh --mode docker -c artex-pg`（连容器内 psql 重写 settings 表的 bcrypt 哈希） |
| 端口被占用 | 启动传参改：`start.bat -addr :9000 -proxy 127.0.0.1:9001`；PG 端口改 `docker run -p` 并同步 config.json |

## 五、在非 Windows 主机上构建的差异

整体流程一致，仅两处不同：

- **第 4 步**：Linux / macOS 下可直接跑官方脚本 `./install.sh` 选「② 本地运行」全自动完成（它内部就是本文档第 2~7 步）；或手动执行 `npm ci && npm run build:static`（POSIX 语法本来就可用）。
- **第 5 步**：官方 `build.sh` 用 `rsync` 同步产物，Linux / macOS 一般自带；Windows 没有 rsync，用本文档的 `rm -rf` + `cp -r`。

## 六、日常运维速查

```bash
start.bat                      # 启动（守护脚本，Ctrl-C 停止）
docker start artex-pg          # 数据库容器被停后手动拉起（带 --restart，正常会自动恢复）
docker logs --tail 50 artex-pg # 查看数据库日志
```

重新构建（改了代码后）：重复第 4 → 5 → 6 步，重启服务；数据与配置不受影响（升级只换程序、不动数据）。
