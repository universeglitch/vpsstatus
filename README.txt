
服务异常恢复与安装失败处理

本项目使用 systemd 管理 Xray 和 Node Monitor Python 面板。Cloudflared 由用户自行安装和配置。

安装脚本 install-dual.sh 支持三种运行模式：

  bash install-dual.sh            正常安装（交互式）
  bash install-dual.sh --check    只读检查当前部署状态，不修改系统
  bash install-dual.sh --recover  在新 Shell 中恢复中断或失败的安装事务

下面各节的命令与脚本实际实现保持一致。


1. 服务异常时如何手动恢复

如果发现某个服务停止或异常，可以单独重启对应服务，不需要重新运行安装脚本。

重启 Xray：

  systemctl restart xray.service

重启 Python 面板：

  systemctl restart node-panel.service

重启 Cloudflared（服务名称以实际安装为准）：

  systemctl restart cloudflared.service

如果提示找不到 Cloudflared 服务，先检查：

  systemctl list-units --all | grep -i cloudflared


2. 查看服务运行状态

分别检查：

  systemctl status xray.service --no-pager -l
  systemctl status node-panel.service --no-pager -l
  systemctl status cloudflared.service --no-pager -l

也可以快速查看三个服务是否正在运行：

  for s in xray node-panel cloudflared; do
    printf "%-16s " "$s"
    systemctl is-active "$s" 2>/dev/null || true
  done

正常运行时应显示 xray active / node-panel active。Cloudflared 由用户自行管理。


3. 查看服务错误日志

查看最近 50 条日志：

  journalctl -u xray -u node-panel -u cloudflared -n 50 --no-pager

实时跟踪某个服务：

  journalctl -u xray -f
  journalctl -u node-panel -f
  journalctl -u cloudflared -f

如果服务启动失败，应先检查日志、确认原因后再处理，不要直接删除配置文件或重新安装。


4. 服务异常自动恢复（自愈）

Python 面板的 systemd 服务由安装脚本写入并已配置：

  Restart=always
  RestartSec=3

即面板进程异常退出后，systemd 会在 3 秒后自动重启。

Xray 的自动恢复通过 systemd drop-in 设置，不改写官方安装器生成的 xray.service：

  /etc/systemd/system/xray.service.d/90-restart-on-failure.conf
  [Service]
  Restart=on-failure
  RestartSec=5s

查看各服务实际的 Restart 配置：

  systemctl show xray -p Restart -p RestartUSec
  systemctl show node-panel -p Restart -p RestartUSec
  systemctl show cloudflared -p Restart -p RestartUSec

注意：安装脚本不会安装、修改或覆盖 Cloudflared 的任何配置，也不会为了设置 Restart
而覆盖用户已有的 systemd 自定义配置。不要对正在主动维护的服务随意执行 stop 或 disable。


5. 设置开机自动启动

检查服务是否已经启用：

  systemctl is-enabled xray.service
  systemctl is-enabled node-panel.service
  systemctl is-enabled cloudflared.service

如需启用：

  systemctl enable xray.service
  systemctl enable node-panel.service
  systemctl enable cloudflared.service

Cloudflared 如果不是通过 systemd 安装的，需要按照实际运行方式配置开机启动。


6. 安装状态检查：bash install-dual.sh --check

--check 为只读模式，只做检查，不会生成密钥、创建目录、修改权限、重启服务或删除任何文件。

它会检查：

  - 系统版本与 CPU 架构
  - 是否检测到 systemd
  - Xray 程序、Xray 配置（JSON 是否有效）、xray.service 单元及其运行/启用状态
  - Python 面板程序、面板配置、node-panel.service 单元及其运行/启用状态
  - auth.json 与 secret.key 是否存在及权限是否为 600
  - Cloudflared 单元（仅查看，不属于本脚本管理范围）
  - /var/backups/node-panel-install-* 下是否存在未完成或回滚失败的事务
  - 是否残留临时文件（如 /run/node-panel-password.*、*.new 等）

运行：

  bash install-dual.sh --check

输出结论为以下之一：

  - 未安装，可进行全新安装
  - 已安装（完整），禁止重复安装
  - 已安装，但服务当前异常
  - 安装中断，存在未完成事务
  - 自动回滚失败，需要人工处理
  - 检测到不完整或不一致的部署状态


7. 安装事务备份目录内容

每次安装都会创建独立的事务目录：

  /var/backups/node-panel-install-<时间戳>.<随机值>/

目录权限为 700，内部包含：

  state.txt      事务状态文件（权限 600），字段包括：
                 transaction_id, created_at, updated_at, status,
                 current_stage, xray_was_active, xray_was_enabled,
                 node_panel_was_active, node_panel_was_enabled
  manifest.tsv   事务清单：每个被管理文件的“真实路径、备份相对路径、安装前是否存在”
  dirs.txt       本次安装新建的目录列表
  meta.txt       记录 Xray 用户/用户组安装前是否存在
  files/         被备份的文件副本

status 取值：

  IN_PROGRESS            安装进行中（正常退出会改为 ROLLED_BACK 或 COMMITTED；
                         kill -9 或断电会一直停留在此状态）
  ROLLBACK_IN_PROGRESS   正在回滚
  ROLLBACK_FAILED        回滚失败，需要人工处理（备份保留）
  ROLLED_BACK            已成功回滚
  COMMITTED              安装成功提交（之后不再自动回滚）
  NOT_STARTED            保留状态

查看最新事务目录：

  ls -ltd /var/backups/node-panel-install-* 2>/dev/null | head


8. 安装脚本中途退出后的处理

install-dual.sh 具有安装事务备份和失败回滚机制，行为取决于退出方式：

  情况 A：脚本以错误码正常退出（例如某一步骤失败）
          由 EXIT trap 自动回滚，恢复安装前状态，并写入 ROLLED_BACK。
  情况 B：收到 Ctrl+C（SIGINT）或 SIGTERM
          尽力自动回滚（这两个信号可被捕获）。
  情况 C：kill -9（SIGKILL）
          Shell 无法捕获，自动回滚不会发生，state.txt 停留在 IN_PROGRESS。
  情况 D：断电 / 系统崩溃
          Shell 无法捕获，自动回滚不会发生，state.txt 停留在 IN_PROGRESS。

无论哪种情况，都必须先人工检查，不要假设脚本已经完成回滚，也不要直接删除文件。

检查步骤：

  bash install-dual.sh --check

  ls -l /usr/local/etc/xray/config.json
  ls -l /usr/local/bin/xray
  ls -l /opt/node-panel/panel.py
  ls -l /etc/node-panel/
  systemctl status xray node-panel --no-pager

如果 --check 提示“安装中断，存在未完成事务”或“自动回滚失败，需要人工处理”，
请使用下一节的 --recover 进行恢复，恢复完成前不要重新安装。


9. 中断事务恢复：bash install-dual.sh --recover

--recover 可以在一个全新的 Shell 中独立运行，不依赖上一次安装进程的任何内存状态，
它完全根据磁盘上的 state.txt / manifest.tsv / dirs.txt / meta.txt 来恢复。

运行：

  bash install-dual.sh --check     # 先检查，确认存在未完成事务
  bash install-dual.sh --recover   # 再恢复

--recover 会：

  - 扫描 /var/backups/node-panel-install-*，列出状态为
    IN_PROGRESS / ROLLBACK_IN_PROGRESS / ROLLBACK_FAILED 的事务，
    并显示创建时间、失败阶段、备份目录
  - 校验备份完整性（清单是否存在、应备份的文件是否都在、状态文件是否可用）
  - 列出安装前服务状态，供你确认
  - 需要你选择事务编号并显式输入 yes 确认后才执行
  - 按该事务的清单恢复文件；只恢复清单中记录的文件，只删除清单中记录为
    “安装前不存在”的文件，并保留权限、属主、时间戳、符号链接属性
  - 恢复服务的启用/运行状态（依据 state.txt 记录）
  - 校验恢复结果，成功后写入 ROLLED_BACK，失败则写入 ROLLBACK_FAILED 并保留全部备份

恢复原则：

  - 不会影响 Cloudflared
  - 不会删除任何备份目录
  - 不是通过 rm -rf 实现
  - 如果备份不完整、状态文件损坏或清单与目标不匹配，会停止并提示人工处理，
    不会猜测或强制恢复


10. 为什么第二次运行安装脚本会提示已有文件？

这是安装脚本的覆盖保护机制。脚本会检查以下项目：

  /usr/local/bin/xray
  /usr/local/etc/xray/config.json
  /opt/node-panel/panel.py
  /etc/node-panel/（auth.json、secret.key、panel-config.json）
  /etc/systemd/system/xray.service 与 xray.service.d
  /etc/systemd/system/node-panel.service
  Xray 用户和用户组
  正在运行的 Xray 进程

只要检测到其中任意安装痕迹，就会停止安装，以免覆盖已有节点、密钥、账号或配置。
脚本会把结果分类为：

  A. 完整安装       已存在完整的部署，禁止重复安装
  B. 部分安装       存在部分安装文件，可能上次中断，请先用 --check / --recover
  C. 残留文件       只有孤立的程序/服务/用户等痕迹且缺少配置，脚本不会自动删除
  D. 未知部署       无法明确归类，为保守起见停止安装，请人工确认

注意：Cloudflared 属于用户外部服务，其缺失或存在都不会被判定为 Xray 安装失败。
“文件存在”不等于“可以删除”，也不等于“可以覆盖”。


11. 安全恢复和重新安装原则

  情况 A：自动回滚成功（state.txt 为 ROLLED_BACK）
          确认服务和文件已恢复到安装前状态后，可以重新运行安装脚本。

  情况 B：自动回滚失败（state.txt 为 ROLLBACK_FAILED）
          先保留备份目录，用 --recover 查看，必要时人工核对，确认恢复后再重新运行。

  情况 C：强制终止或断电（state.txt 停留在 IN_PROGRESS）
          先执行 --check，再执行 --recover，不要直接删除文件。

严禁为了绕过覆盖保护而直接执行：

  rm -rf /usr/local/etc/xray
  rm -rf /etc/node-panel
  rm -rf /opt/node-panel

这些目录可能包含正在使用的节点配置、面板认证信息和密钥。

核心原则：先检查（--check）、再恢复（--recover）、最后重新安装。
不要用删除文件的方式绕过安装保护。








Node Monitor Panel Backup
=========================

Contents:
1. panel.py
   Current panel source code.

2. node-panel.service
   Clean systemd service template without credentials.

IMPORTANT SECURITY NOTES:
- This archive does not contain auth.json.
- This archive does not contain secret.key.
- This archive does not contain Xray configuration.
- This archive does not contain Cloudflare Tunnel credentials.
- Existing login credentials and sessions cannot be restored from this archive.

INSTALLATION:
1. Create the directory:
   mkdir -p /opt/node-panel

2. Copy panel.py:
   cp panel.py /opt/node-panel/panel.py

3. Install the service template:
   cp node-panel.service /etc/systemd/system/node-panel.service

4. Ensure the authentication files are separately configured:
   /etc/node-panel/auth.json
   /etc/node-panel/secret.key

5. Reload and start:
   systemctl daemon-reload
   systemctl enable node-panel
   systemctl restart node-panel

6. Check service status:
   systemctl status node-panel

NOTE:
The service template assumes the panel runs as root and uses
/usr/bin/python3. Verify the paths and authentication files
before restoring on another machine.

#installation
use install.sh
