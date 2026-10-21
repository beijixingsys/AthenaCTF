# Athena安全竞赛平台

  AthenaCTF 竞赛平台是一套面向网络安全竞赛的自托管一体化赛事系统，无需额外数据库和复杂中间件，部署轻量、开箱即用。平台覆盖赛事运营全流程：管理员可管理赛事、题库、附件、容器模板、队伍与选手账号及角色权限；选手端提供赛事浏览、题目详情、附件下载、Flag 提交、实时排名与个人中心。计分引擎支持每队每题一次计分、一血奖励、可选前十加分与人工调分留痕，全部操作事务保护，成绩榜真实可追溯。平台内置四种赛制：传统解题赛（动态靶机按队独立启停，Flag 加密随机生成、逐队隔离、到期自动回收）；AI 智能体解题赛（队伍 Token 接入 REST API，供 AI Agent 自动答题，支持动态衰减计分）；AWD-PLUS 攻防赛（每队独立靶机、离线修补包上传校验、攻防双向动态计分与轮次自动结算）；理论竞赛。此外还提供全屏比赛大屏，实时展示排行榜、解题动态与播报，适合赛场投屏。
安全与运维方面，平台提供 Docker 靶机资源限制（CPU/内存/PID）、非 root 运行与权限收敛、请求限速、敏感提交脱敏、实例到期回收与重启恢复、一致性备份等机制，可在 Windows 与 Linux 上跨平台运行。

<img width="2559" height="1271" alt="image" src="https://github.com/user-attachments/assets/82bc0662-f63c-4a87-946a-e6963c63efcb" />

## 一、Windows 部署

1. 下载并解压 Windows x64 发布包，例如解压到 `D:\AthenaCTF`。
2. 双击 `AthenaCTF.exe`，保持服务窗口运行。
3. 浏览器打开管理端：**http://127.0.0.1:10012**。
4. 按页面引导创建管理员，保存随机生成的初始密码。
5. 选手端地址：**http://127.0.0.1:10011**。
6. 停止平台时，在服务窗口按 **Ctrl+C**，等待程序退出。

平台程序支持 Windows 10 / Windows 11 x64，无需安装 Go、Node.js 或数据库。请将程序放在有写入权限的目录，不要直接在压缩包内运行。

## 二、Linux 部署

x64 主机执行：

```bash
tar -xzf AthenaCTFv1.5-linux-amd64.tar.gz
cd AthenaCTFv1.5-linux-amd64
chmod +x AthenaCTF
./AthenaCTF
```

ARM64 主机使用 `linux-arm64` 包，并将命令中的目录名改为对应名称。

在服务器本机打开 **http://127.0.0.1:10012** 完成管理员初始化。没有桌面环境时，可以在自己的电脑建立 SSH 隧道：

```bash
ssh -L 10012:127.0.0.1:10012 用户名@服务器IP
```

保持 SSH 连接，在自己电脑的浏览器打开 **http://127.0.0.1:10012**，创建管理员并保存随机密码。

## 三、开启局域网访问

首次启动后，程序会在可执行文件所在目录生成 `config.env` 和 `data/`。

1. 停止平台。
2. 打开 `config.env`，将 `CTF_BIND_HOST=127.0.0.1` 改为 `CTF_BIND_HOST=0.0.0.0`。
3. 重新启动平台。
4. 在服务器防火墙放行 TCP **10011**（选手端）和 **10012**（管理端），按需要限制访问来源。
5. 其他电脑访问 `http://服务器IP:10011` 或 `http://服务器IP:10012`。

管理端初始化完成后再开启网络访问。若用于公网，需另行配置域名、HTTPS 反向代理和访问控制。

## 四、修改端口和数据位置

编辑程序旁的 `config.env`，保存后重启：

```ini
GIN_MODE=release
TZ=Asia/Shanghai
CTF_BIND_HOST=0.0.0.0
CTF_APP_PORT=10010
CTF_USER_WEB_PORT=10011
CTF_ADMIN_WEB_PORT=10012
CTF_DB_PATH=data/ctf_platform.db
CTF_UPLOAD_ROOT=data/uploads
CTF_INSTANCE_HOST=auto
CTF_NETWORK_POLICY_READY=0
```

- 三个端口必须互不相同，且未被其他程序占用。
- API 默认使用 10010；网页通过自身端口转发 API，一般不需要单独对外开放 10010。
- 相对数据路径以可执行文件所在目录为起点。
- 系统环境变量优先于 `config.env`。文件修改未生效时，检查是否设置了同名环境变量。
- 不要同时启动两份程序共用同一数据库。

## 五、配置容器题运行环境

理论赛、静态附件题和平台管理不需要 Docker。容器题需要 **Docker CLI 和 Linux 容器引擎**。

1. 为服务器安装合适的 Docker 环境。Windows 请使用支持当前系统版本的 Docker 环境，并切换到 Linux 容器。
2. 使用运行平台的同一用户执行 `docker version`，确认能连接引擎。
3. 如果连接指定引擎，在 `config.env` 增加 `CTF_DOCKER_HOST=引擎地址`；不要将未认证的 Docker API 暴露到公网。
4. Linux 容器宿主机的网络隔离配置参考 `deploy/docker-daemon.json.example` 和 `deploy/firewall.sh`。先查看文件内容，根据实际网段和管理端口配置，再部署并验证隔离。
5. 确认题目容器不能越权访问管理服务和其他队伍后，设置 `CTF_NETWORK_POLICY_READY=1`，重启平台。
6. 在管理端检查 Docker 状态，再按 `docs/CHALLENGE_IMPORT.md` 导入题目。

`CTF_NETWORK_POLICY_READY=1` 仅表示管理员已确认隔离，不能代替实际防火墙配置。

## 六、Linux 设置开机启动

将程序放入 `/opt/athenactf`，创建专用运行用户并赋予目录权限：

```bash
sudo useradd --system --home /opt/athenactf --shell /usr/sbin/nologin athena
sudo chown -R athena:athena /opt/athenactf
```

用户已存在时无需重复创建。若需要 Docker，确认该用户可通过 Docker CLI 访问配置的引擎。

新建 `/etc/systemd/system/athenactf.service`：

```ini
[Unit]
Description=Athena CTF Platform
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=athena
WorkingDirectory=/opt/athenactf
ExecStart=/opt/athenactf/AthenaCTF
Restart=on-failure
RestartSec=5
KillSignal=SIGINT
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```

先停止手动运行的进程，再执行：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now athenactf
```

常用命令：

```bash
sudo systemctl status athenactf
sudo systemctl restart athenactf
sudo systemctl stop athenactf
journalctl -u athenactf -f
```
## Athena竞赛平台交流群
<img width="351" height="495" alt="image" src="https://github.com/user-attachments/assets/24ec5cab-8679-42f6-9942-eed73b9b0e6e" />

## 支持我们
<img width="1152" height="1152" alt="image" src="https://github.com/user-attachments/assets/49cb8492-34d4-4503-95c6-b5919f45bde8" />

若本项目帮助到您请赞助本项目，您的支持不仅是鼓励，更是推动项目不断精进的重要力量！
