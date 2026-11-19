# GoFileBrowser 用户指南

轻量级 Web 文件浏览、搜索与预览服务，面向单机共享目录。可免费使用。

## 1. 能力概览

- 递归索引共享目录，启动差分扫描 + 文件变更监控；
- 按文件名、目录名、扩展名搜索，支持 `type:`、`case:`、引号短语；
- 目录浏览、面包屑、名称/大小/时间排序、分页；
- 图片、PDF、音视频、文本、Markdown 在线预览；
- 文件下载、断点续传、ETag 缓存；
- MD5 / SHA256 查询，结果缓存在索引里；
- 目录打包下载：ZIP 或 TAR.GZ；
- 隐藏路径、符号链接逃逸、特殊文件读取的保护。

## 2. 快速开始

### 2.1 直接运行发行包

```bash
tar -xzf GoFileBrowser-<version>-<commit>-linux-amd64.tar.gz
cd GoFileBrowser-<version>-<commit>-linux-amd64

# 第一次运行会生成 etc/config.toml
./GoFileBrowser --public /srv/share --listen :8081
```

浏览器打开：

```text
http://<服务器IP>:8081/
```

> 启动时会校验共享目录：必须存在且是目录；启用索引时索引目录不能位于共享目录内。

### 2.2 配置

编辑 `etc/config.toml`：

```toml
[server]
listen = ':8081'

[public]
share_directory = '/srv/share'
search_index_enabled = true
search_index_directory = 'data/index'
search_index_watch_enabled = true
```

常用启动参数：

```bash
./GoFileBrowser --public /srv/share --listen :8081
./GoFileBrowser --help
```

配置优先级：

```text
etc/config.toml  <  GOFILEBROWSER_* 环境变量  <  命令行参数
```

环境变量：

| 变量 | 说明 |
| --- | --- |
| `GOFILEBROWSER_LISTEN` | 监听地址，如 `:8081` |
| `GOFILEBROWSER_PUBLIC` | 共享目录 |
| `GOFILEBROWSER_INDEX_ENABLED` | 是否启用索引 |
| `GOFILEBROWSER_INDEX_DIR` | 索引目录 |
| `GOFILEBROWSER_INDEX_WATCH` | 是否监控文件变化 |
| `GOFILEBROWSER_LOG_LEVEL` | `debug` / `info` / `warn` / `error` |
| `GOFILEBROWSER_LOG_FILE` | 日志文件路径 |

### 2.3 systemd

```bash
sudo useradd --system --home /opt/gofilebrowser --shell /usr/sbin/nologin gofilebrowser
sudo mkdir -p /opt/gofilebrowser /srv/gofilebrowser/share /srv/gofilebrowser/index
sudo tar -xzf GoFileBrowser-<version>-<commit>-linux-amd64.tar.gz \
  --strip-components=1 -C /opt/gofilebrowser
sudo cp /opt/gofilebrowser/etc/config.example.toml /opt/gofilebrowser/etc/config.toml
sudo chown -R gofilebrowser:gofilebrowser /opt/gofilebrowser /srv/gofilebrowser
```

编辑配置后安装参考 unit：

```bash
sudo cp /opt/gofilebrowser/deploy/gofilebrowser.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now gofilebrowser
```

也可用程序生成 unit：

```bash
cd /tmp
sudo /opt/gofilebrowser/GoFileBrowser gen-systemd \
  --listen :8081 --public /srv/gofilebrowser/share
sudo mv /tmp/GoFileBrowser.service /etc/systemd/system/gofilebrowser.service
```

### 2.4 Docker / Compose

Docker 构建需要源码仓库（`Dockerfile`、`go.mod`、`static/`、`templates/`），发行包适合直接运行或 systemd。

```bash
mkdir -p share
docker compose up -d --build
```

容器内共享目录挂载到 `/data/share`，索引/日志使用命名卷。

## 3. 使用说明

### 3.1 搜索

- 输入即搜索，默认每页 50 条，最多 200 条/页；
- 单次查询最多翻 10,000 条；超出会提示细化关键词；
- 多个词是 **OR** 关系；
- `type:image`、`type:audio`、`type:video` 按类型过滤；其余按扩展名，如 `type:pdf`、`type:zip`、`type:iso`；
- `case:sensitive` 区分大小写；
- 引号包住短语，例如 `"annual report"`；
- 目录名也能命中，结果可直接打开目录，但不会展开目录下的全部子项。

### 3.2 浏览与预览

- 目录行：`打开` + `…` 菜单；
- 文件行：`下载` + `…` 菜单；
- 文件菜单：预览、下载、复制下载链接、计算 MD5、计算 SHA256；
- 目录菜单：下载为 ZIP、下载为 TAR.GZ；
- 预览支持：图片、PDF、音视频、文本、Markdown；
- 不支持在线预览的类型请下载后查看。

### 3.3 Markdown

Markdown 预览支持 CommonMark + GFM：

- 标题、列表、引用、代码块；
- 表格、任务列表、删除线、自动链接；
- 脚注、定义列表；
- 相对图片/文档链接会解析到共享目录内；
- 原始 HTML、数学公式、Mermaid、代码高亮暂不支持；
- 源文件超过 2 MiB 时请下载查看。

### 3.4 下载与 hash

- `/download/...` 支持 `HEAD`、`ETag`、`Range`；
- 目录打包：`/archive/<目录>?format=zip` 或 `?format=tgz`；
- hash 结果按“文件 + 算法”缓存在索引中；
- 文件修改后下一次 hash 会重新计算；计算期间文件变化会返回 409，请重试。

### 3.5 目录压缩包

- ZIP：通用格式，Windows/macOS/Linux 都可直接打开；
- TAR.GZ：Linux 场景更常用；
- 隐藏路径、符号链接、FIFO/设备文件不会进入压缩包；
- 同一时间最多 2 个打包任务；
- 压缩包是流式生成的，不在服务器落临时大文件。

## 4. 安全建议

- 当前版本**没有内置登录认证**，不要直接暴露到公网；
- 公网使用请放在 Caddy/Nginx/VPN 后面，并加认证与限速；
- 建议使用独立系统用户运行，不要把共享目录设为全局可写；
- 共享目录内的隐藏路径、越界符号链接、特殊文件会被拒绝或跳过；
- 目录打包会消耗 CPU/带宽，建议在代理层限制并发和下载速度。

反向代理示例（Caddy）：

```caddyfile
files.example.com {
    reverse_proxy 127.0.0.1:8081
}
```

## 5. 维护与升级

- 日志：`logs/app.log`；
- 索引是派生数据，可重建；
- 停止服务后替换二进制、`static/`、`templates/`，再启动；
- 升级前备份 `etc/config.toml` 和 `data/index`；
- 如果索引异常，先保留旧 `data/index` 再重建，不要直接删除用户文件。

## 6. 故障排查

**启动提示“共享目录不可访问/不是目录”**

检查 `share_directory`，确保路径存在并且是目录。

**启动提示“索引目录不能位于共享目录内”**

把 `search_index_directory` 移到共享目录之外。

**端口被占用**

```bash
ss -ltnp | grep 8081
```

修改 `--listen` 或配置里的 `listen`。

**页面没有样式/操作菜单无反应**

强制刷新浏览器；确认 `static/` 与程序在同一目录，且反向代理没有拦截 `/static/`。

**hash 返回 409**

文件在计算过程中被修改，重试即可。

**Markdown 提示渲染失败/文件过大**

确认文件小于 2 MiB；当前不支持原始 HTML、公式和 Mermaid。

## 7. 使用与分发

本软件以 Apache License 2.0 授权，可免费使用、修改和分发；完整条款见包内 `LICENSE`，项目声明见 `NOTICE`。

第三方组件（Go 模块、Go runtime、htmx、SweetAlert2 等）遵循各自许可证，完整清单与许可文本见 `THIRD_PARTY_LICENSES.md` 和 `third_party/licenses/`。再分发时请保留这些声明。
