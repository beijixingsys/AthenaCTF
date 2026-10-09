# Athena安全竞赛平台

提供 Windows x64、Linux x64、Linux ARM64 可执行部署包。所有包均不包含当前题库、题目附件、题目镜像、现有账号、队伍、成绩或数据库。首次运行自动创建空数据库，创建首位管理员时由系统生成随机密码，保存后再添加赛事、队伍和自己的题目。

## 1. 选择下载包

| 文件名中的标识                    |                                                              |
| --------------------------------- | ------------------------------------------------------------ |
|                                   |                                                              |
| `windows-amd64-no-questions.zip`  | Windows 10 1809+ / Windows 11 x64 运行包（平台本体不要求 Docker Desktop）。 |
| `linux-amd64-no-questions.tar.gz` | Intel/AMD x64 Linux 服务器运行包，推荐正式比赛使用。         |
| `linux-arm64-no-questions.tar.gz` | ARM64 Linux 服务器运行包。题目镜像也必须支持 ARM64。         |
| `SHA256SUMS.txt`                  | 压缩包 SHA-256 校验值。                                      |

运行包已经包含 Go 可执行程序与两个前端生产页面，新服务器**无需安装 Go、Node.js、npm、MySQL**。数据库是程序自带的 SQLite 驱动。Docker 用于后续添加的容器题。

解压到固定目录后运行脚本。Linux 建议 `/opt/bjxctf`，路径必须是无空格的 ASCII 路径，且服务账号能遍历其上级目录，不要放到 `/root` 下。不要在压缩包预览窗口里直接运行，也不要混合不同系统的文件。每个运行包内的 `MANIFEST.json` 记录逐文件摘要，`BUILDINFO.json` 记录编译架构。

## 2. 推荐部署系统和资源

正式比赛推荐 **Ubuntu Server 24.04 LTS + 原生 Docker Engine + systemd**。Linux 下 Docker 直接使用宿主机内核，后台服务、重启恢复和网络隔离更容易统一管理；Windows 更适合开发、演示和日常测试。Docker 官方支持在 Ubuntu 上通过签名软件源安装 Engine；Docker Desktop 不支持 Windows Server，因此不要在 Windows Server 上运行本包的 Desktop 安装流程。[Docker Ubuntu 安装说明](https://docs.docker.com/engine/install/ubuntu/)、[Docker Windows 系统要求](https://docs.docker.com/desktop/setup/install/windows-install/)

针对约 100 人、30 队、每队同时 1 个 Web 容器，建议先准备 **16 vCPU、64 GB 内存、500 GB NVMe SSD、千兆网络**。轻量题、单容器内存限制约 512 MB 时可用 8 vCPU/32 GB 起步；若单题需要 1 GB 或更多，应按 30 个实际镜像峰值重新估算。以上是容量规划建议，不替代题目组合的 30 并发验收。保留数据库、镜像、备份空间，并将备份保存到另一块磁盘或另一台机器。

## 3. Linux 一键部署

自动安装覆盖 Ubuntu 22.04/24.04、Debian 12/13，要求 systemd、sudo/root 权限、可访问发行版软件源。Docker 安装需要 `download.docker.com`（可选，不影响平台本体）。脚本不支持通过 Docker Desktop 管理 Linux，也不会自动卸载已有 Docker 或覆盖已有数据。

示例（替换为实际包名）：

```bash
sha256sum -c SHA256SUMS.txt
tar -xzf AthenaCTFv1.2-linux-amd64-no-questions.tar.gz
cd AthenaCTFv1.2-linux-amd64-no-questions
sudo bash deploy.sh --check
sudo bash deploy.sh
```

部署脚本检查架构、系统、必需命令并创建平台 systemd 服务。Docker 仅用于容器题：已安装则保留，未安装时交互确认后从官方签名源安装，也可用 `--no-docker` 跳过或安装失败继续。**缺少 Docker 不影响平台管理、空库初始化或静态题**，仅容器题不可用。配置已存在时保留现有 `config.env` 和 `data/`。包应放置在固定服务器路径，部署后不要随意移动。

首次安装默认绑定 `127.0.0.1`，避免空库首位管理员初始化接口在局域网上被他人抢先使用。部署完成后**直接在服务器终端**完成首次管理员初始化（输入账号与昵称，系统生成随机密码，仅在当前终端显示一次，不进入服务日志或命令行参数）：

初始化完成后，在服务器终端执行：

```bash
sudo bash deploy.sh --publish
```

脚本会先确认管理员已创建，再将平台绑定改为 `0.0.0.0`，支持双网卡上的服务器地址。若已自行修改端口，SSH 隧道也要使用对应端口。防火墙/云安全组仅向比赛网络和管理网络放行需要的入口；脚本不改写你原有 SSH 和全局防火墙策略。

后续启动、关闭：

```bash
sudo bash start.sh
sudo bash stop.sh
```

编辑 Linux `config.env` 后使用 `sudo bash start.sh --restart` 重新加载。`--check` 是只读检查，缺少依赖时返回非零退出码；执行普通 `deploy.sh` 才会安装。Linux 部署后平台和 Docker 配置为随主机开机自启，关闭脚本不会取消这个设置。

**`stop.sh` 默认只停止本包的平台服务**，不会停止 Docker、任何容器或其他应用。停止后平台端口关闭；数据、上传文件和配置保留。**`stop.sh --docker`** 是显式的整机关闭维护入口，会停止本机 Docker Engine 及其上**所有**容器（包括其他应用的），仅在比赛专用主机上使用。下次启动用 `sudo bash start.sh`。题目实例是否继续有效由平台的过期时间与恢复逻辑决定，不应假设停机等于暂停赛程。

## 4. Windows 一键部署

平台本体支持 Windows 10 1809+ 和 Windows 11 x64，PowerShell 5.1+；不需要 Docker 即可启动/初始化/管理平台和跑静态题。容器题需要 WSL 2 + Docker Desktop（仅 Win10 22H2 19045 和 Win11 22631+ 有可验证支持）。解压包到例如 `D:/bjxctf`。

1. 双击 `check.cmd`，区分平台可运行 / 容器引擎未就绪 / 真不支持，只读不改动。
2. 双击 **deploy.cmd**，按 UAC 提示授权管理员权限（仅用于写本包目录；不会安装 WSL 或 Docker）。
3. 脚本部署平台并后台启动 API、选手端、管理端；无 Docker 也正常启动和初始化。
4. 本机浏览器访问 `http://127.0.0.1:10012`，创建管理员并下载/保存系统生成的随机密码；确认保存后进入管理端，随后双击 `publish.cmd` 开启全部网卡访问。
5. 容器题需要 Docker（可选）：双击 `install-containers.cmd`（显式 opt-in），或在 `config.env` 中配置 `CTF_DOCKER_HOST` 指向已有引擎。仅在 build 支持范围内自动安装。

容器支持的 BIOS/UEFI 虚拟化、Docker 安装许可和系统重启由部署方手动完成（仅容器题需要）。平台的启动、初始化和管理不依赖任何容器引擎。“一键部署”负责检测、安装和启动；遇到这些系统要求时会明确提示下一步，不会假装安装成功。Docker Desktop 的适用许可由部署方确认。[Docker Desktop Windows 安装文档](https://docs.docker.com/desktop/setup/install/windows-install/)

启动/停止本包平台。默认 `stop.cmd` 仅停止本包平台服务，不影响 Docker 或其他应用容器；如需显式关闭整个 Docker Engine（影响同机其他应用），另用 `stop-docker.cmd`。

Windows 包的启动器使用本包相对路径，不依赖原开发机的 `D:\2026\...` 或 `.runtime/current-backend.json`。脚本采用 ASCII 文本，兼容 Windows PowerShell 5.1，避免中文编码造成解析错误。

## 5. 地址、题目与环境隔离

发布后：选手端 `http://<服务器IP>:10011`，管理端 `http://<服务器IP>:10012`，API `http://<服务器IP>:10010/api/health`。同机 Docker 的 `CTF_INSTANCE_HOST=auto` 让题目地址使用选手访问平台时的服务器 IP。独立 Docker 节点需另行配置节点地址和访问权限，本次一键脚本用于同机 Docker。

本包没有任何预置赛题或题目镜像，不会拉取当前平台比赛的镜像。你需要在管理端新建题目、上传新附件，构建/导入自己准备的镜像并配置容器模板。平台导入适配器代码和人工创建题目的功能完整保留，旧题库正文/附件/答案未提供。新赛题镜像需要遵守平台 `FLAG` 环境变量注入约定。

`CTF_NETWORK_POLICY_READY=0` 是新包默认值：网页、管理、静态题判题可运行，容器启动暂不放行。部署方要按 `docs/DEPLOYMENT.md`、`deploy/firewall.sh` 在真实题目节点配置并验证网络隔离后，再设置为 `1` 并重启平台。仅安装 Docker 不等于容器网络已经隔离。现有防火墙例子面向 Docker iptables 后端，应先核对实际后端、地址池和网段。

如果需要 HTTPS，参考包内 `deploy/nginx.conf.example` 配置反向代理；配置 `CTF_PUBLIC_BASE_URL` 和受信代理范围。按最终域名或各网卡地址完成选手登录、容器连接、提交、计分、榜单和大屏验证后再正式开赛。

## 6. 目录、备份与升级

| 新运行包内路径                                 | 含义                                       |
| ---------------------------------------------- | ------------------------------------------ |
| `bin/`、`web/`                                 | 平台程序、管理/选手前端，必须保留。        |
| `config.env`                                   | 本机配置，必须保留，不随新包覆盖。         |
| `data/ctf_platform.db`、可能存在的 `-wal/-shm` | 真实数据库；禁止运行中只拷贝主文件。       |
| `data/uploads/`                                | 后续上传的附件、封面等，与数据库配套备份。 |
| `logs/` 或 Linux journal                       | 运行日志，可按保留策略清理。               |
| `run/`                                         | PID/部署状态，用于启动停止，不是比赛数据。 |
| `install-cache/`（Windows）                    | 已下载安装器，安装完成后可删除。           |

先停平台再复制整个 `data/`（包含数据库、uploads 和 AWD-PLUS 修补包 vault/staging）和 `config.env`，或使用 `scripts/backup.py` 的 SQLite 一致备份流程并手动补存 vault/staging 目录。当前主机的 `.runtime` 另见 `RUNTIME_STORAGE.md`，不能按新运行包的日志目录对待。升级先备份业务数据，替换程序和前端，再启动并检查迁移结果；不要将本次空库部署包覆盖到原机的数据库上。

源码重建：准备 Go 1.24+、Node.js 24、Python 3.11+，在两个前端目录执行 `npm ci`，回到源码根目录执行：

```text
python scripts/build_clean_distribution.py --stage all --version YOUR-NEW-VERSION
```

请使用 `build_clean_distribution.py`。历史 `build_distribution.py` / `deploy/build.sh` 是以前含题库交付的工具，其规则与本次交付不同。

旧开发机维护脚本（引用旧 .runtime 路径）已从此源码包排除。新机请使用运行包根目录中的部署、启动和关闭脚本。

## 7. 验证边界

交付前会重新编译可执行文件与双前端，检查压缩包清单、摘要和禁止目录，并使用隔离临时目录启动 Windows 版验证三端口及空库初始化。Linux 文件可验证 ELF 架构和 shell 语法；当前 Windows 开发机不能代替一台全新的 Linux/Windows 主机验证系统包安装、UAC、重启、BIOS、外网软件源与实际网络 ACL。不要将这些环境相关项视为已经在新主机验收。

交付目录中的 `VERIFICATION.json` 记录实际执行的项目和结果；不存在真实新主机安装测试时会明确标记，不能用构建成功替代部署成功。
