# 远程操作手册：Mac 通过 UU远程端口映射 + SSH 操控 Windows

> **什么时候用**：字节码（.jsc）的加载、试跑、编译必须在 x86-64 Windows + 同款 V8 运行时里做，而 AI 不在那台 Windows 上。打通这条通道后，传文件、跑脚本、查日志、装软件全部自动化，"AI 写命令 → 用户复制粘贴 → 截图回传"的人工传话筒模式退役。
> **先用 SKILL.md §7 的环境判断表选场景**：AI 与目标软件同处一台 Windows 时**整章跳过**；两台不同 Windows 时本章思路照用、客户端命令换 Windows 自带 OpenSSH；AI 在 Mac（本 Skill 实战场景）才需要完整修这条通道。

---

## 1. 通道拓扑（一句话）

```
Mac（AI 所在地）──ssh -p 10022 administrator@127.0.0.1──► UU远程端口映射（Mac 10022 → Windows 22）
      ◄────────────────结果原路返回────────────────── Windows 实体机 sshd 执行命令
```

**语义**：UU远程（国产远程桌面软件）的"端口映射"功能，在 Mac 本地监听一个端口（如 10022），把进来的连接转发到 Windows 侧的指定地址端口（`127.0.0.1:22`）。配置表单字段：**名称** `ssh`、**本地端口** `10022`（Mac 侧）、**目标地址** `127.0.0.1`、**目标端口** `22`（Windows 侧）、勾选「保存后立即启用」。
**记住**：本地端口 = Mac 侧监听端口；目标地址/端口 = Windows 侧回环地址。

> UU 的两个坑：① 远程窗口被遮挡时键盘焦点传不到 Windows——先 activate_window 提到最前再打字；② UU 的「终端」按钮实测只打开远程桌面而非终端，别指望它，正解是把 sshd 修好拿全功能通道。

---

## 2. 建立通道：sshd 修复全链

Windows 自带 OpenSSH 服务常常是坏的（案例里二进制日期停在 2019 旧版，典型的"系统优化软件搞坏系统组件"）。完整修复链：
**在哪执行**：本节 PowerShell 命令在 **Windows 侧的管理员 PowerShell** 里跑（UU远程盯屏或本机操作都行），对工作目录无要求——命令全部用绝对路径，不依赖当前位置。

### 2.1 诊断三件套（先拿真错误码，别瞎试）

```powershell
# ① 二进制 sanity check：活没活着
$exe="C:\Windows\System32\OpenSSH\sshd.exe"; Get-Item $exe | Select-Object Length,LastWriteTime; & $exe -V; "exitcode=$LASTEXITCODE"
# ② 事件日志拿精确错误码（反复报 7000 服务启动失败 / 7009 三十秒超时 = sshd 损坏的典型组合）
Get-WinEvent -FilterHashtable @{LogName="System"; ProviderName="Service Control Manager"} -MaxEvents 6 -ErrorAction SilentlyContinue | Format-List TimeCreated,Id,Message
# ③ 单连接调试模式看前台表现
& $exe -d
```

案例实测：`sshd.exe -d` 静默秒退无输出 → 二进制连初始化都做不到；`exitcode=-1073741511`（0xC0000139 = STATUS_ENTRYPOINT_NOT_FOUND，依赖 DLL 入口点缺失）。
**排查心法：先用 `-V` 做二进制 sanity check，再查 SCM 事件日志拿错误码。**

### 2.2 失败方案（按尝试顺序，别再走）

1. `Remove-WindowsCapability + Add-WindowsCapability` 重装——拉回**同一份坏文件**，无效。
2. `Invoke-WebRequest` 下载官方包——文件被写坏，解压报"找不到中央目录结尾记录"。**下载坑两连**：① 让用户"只粘一次"，命令开头先 `Remove-Item` 删旧文件（用户把同一条命令粘两次、并发写同一文件会把 zip 写坏）；② curl 直接重下报 `CRYPT_E_REVOCATION_OFFLINE`——国内网络够不到证书吊销服务器。
3. `sc.exe config binpath` 指向新二进制——服务从"30 秒超时"变"秒拒"，快速失败：根因不是路径，是宿主密钥权限（见下）。

### 2.3 成功方案（两步，照抄）

**第 1 步：curl 下载（正法）**
```powershell
Remove-Item "$env:USERPROFILE\Downloads\ossh.zip" -Force -ErrorAction SilentlyContinue
curl.exe -L --ssl-no-revoke --retry 3 -o "$env:USERPROFILE\Downloads\ossh.zip" "https://github.com/PowerShell/Win32-OpenSSH/releases/latest/download/OpenSSH-Win64.zip"
```
`--ssl-no-revoke` 是 schannel 后端专用选项：保留证书校验、只跳过吊销检查——比 `-k`（全放弃校验）干净。

**第 2 步：解压 + 官方脚本重建服务**
```powershell
$D="C:\Program Files\OpenSSH"; Expand-Archive -Force "$env:USERPROFILE\Downloads\ossh.zip" $D
sc.exe stop sshd | Out-Null; sc.exe delete sshd
powershell -NoProfile -ExecutionPolicy Bypass -File "$D\OpenSSH-Win64\install-sshd.ps1"
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic    # ★ 设自启动：本 Skill 把通道当长期资产用，Windows 重启后通道不能断
```
通过标准：`install-sshd.ps1` 输出 `Repaired permissions` → `sshd and ssh-agent services successfully installed` → `Status: Running`，22 端口同时 Listen；再跑 `Get-Service sshd | Select-Object Status,StartType` 验收——必须是 `Running` + `Automatic`。**只 Running 不算过**：`StartType = Manual` 意味着 Windows 一重启通道即断，等于白修。

**根因**：新 sshd 前台 `-d` 跑得欢（Server listening on ::/0.0.0.0:22），服务模式却失败——**host key 私钥 ACL 违反 StrictModes**：Administrator 账户对 `C:\ProgramData\ssh\ssh_host_*_key` 有访问权，服务以 LocalSystem 运行时被拒。install-sshd.ps1 的 FixHostKeyPermissions 修的正是这件事。**以后遇到"前台行、服务不行"的 sshd，直接跑 install-sshd.ps1，别手工折腾 ACL。**

**知识点**：`sshd -d` 是单连接调试模式（接待一个连接就退出）；`sshd -D` 是持久前台模式。要经映射反复连，必须用 `-D` 或服务模式。单连接限制下的 scp 变通：`ssh ... "cat > dest.zip" < local.zip`（stdin 走同一条会话）。

### 2.4 干净系统装 OpenSSH 的最小命令链（如果系统组件没坏）

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Set-Service -Name sshd -StartupType Automatic
Start-Service sshd
New-NetFirewallRule -DisplayName 'OpenSSH...'   # 放行 22 端口
```
注意：`Add-WindowsCapability -Online` 需要联网下载——在无人值守安装的网络未就绪阶段会静默失败，挪到首登阶段执行。

---

## 3. Mac 侧验证三连（映射配好后）

```bash
nc -z -w 3 127.0.0.1 10022 && echo PORT_OPEN          # ① 映射活没活
timeout 5 bash -c 'exec 3<>/dev/tcp/127.0.0.1/10022 && head -c 40 <&3' | cat -v   # ② 对端是不是 SSH（看 SSH-2.0 横幅）
ssh -p 10022 -o ConnectTimeout=5 -o StrictHostKeyChecking=no -o BatchMode=yes test@127.0.0.1 exit  # ③ 实测
```
② **不能写 `head -c 100`**（会假死约 1 分钟）：OpenSSH 横幅只有 ~56 字节，且服务器发完横幅不主动断开连接，`head` 等不满 100 字节会一直挂到对端闲置超时（实测阻塞 55–60 秒）。交互执行时极易被误判成"映射假连通"然后瞎重试。**判读**：几秒内打印 `SSH-2.0-...` = 对端真是 SSH；5 秒被 `timeout` 杀掉、或打印的不是 SSH 字符串 = 22 端口上没有 SSH 服务（回去修 sshd）。
症状对照："TCP 通但对端在 SSH-banner 前就关闭" = Windows 22 端口没有服务在听（回去修 sshd）。
**记住：端口通 ≠ 服务可用，检测要看协议横幅，不检测空端口。**

### 密钥与授权

- Mac 侧生成专用密钥：`ssh-keygen -t ed25519 -N '' -C 'decard-probe' -f <工程目录>/uu-remote/probe_key`
- Windows 侧授权**两处都要写**（否则管理员登录被拒）：
  - `%USERPROFILE%\.ssh\authorized_keys`
  - `C:\ProgramData\ssh\administrators_authorized_keys`（管理员组账户专属文件）

### 日常命令模板

```bash
# 执行单条命令（大段脚本加 -tt 防假死）
ssh -i <key> -p 10022 -o ConnectTimeout=8 -o StrictHostKeyChecking=accept-new -o BatchMode=yes administrator@127.0.0.1 "<cmd>"
# 传文件（目标路径用正斜杠；大文件走后台任务 + 完成通知，别前台干等）
scp -i <key> -P 10022 -o BatchMode=yes <local> administrator@127.0.0.1:C:/Temp/
```

### 文件位置约定（立死规矩）

| 用途 | 位置 |
|---|---|
| 传过去的文件 | `D:\下载`（或 `C:\Users\Administrator\Downloads`） |
| 安装的软件 / 部署目录 | `D:\软件\<目标软件>自用版`（独立目录名，避开原版） |
| 探针/编译工作目录 | `C:\Users\Administrator\AppData\Local\Temp\<名字>-probe` 等绝对路径 |
| 日志回收 | 脚本写 ndjson 到绝对路径 → scp 拉回 Mac 读 |

---

## 4. PowerShell EncodedCommand 黄金范式（整个远程操作的地基，背下来）

**为什么需要**：命令行嵌中文/引号/`$`变量，经 ssh+cmd 层层转义后必坏（多层转义地狱）；远程输出混 XML 标记（CLIXML）和 GBK 乱码，机器解析不了。

**范式**：
```bash
# Mac 侧：写 .ps1（可含中文、多行、引号随便写）
cat > /tmp/fix_sshd.ps1 <<'EOF'
... PowerShell 脚本 ...
EOF
# 转码：EncodedCommand 要求 UTF-16LE 的 base64（不是 UTF-8！）
B64=$(iconv -f UTF-8 -t UTF-16LE /tmp/fix_sshd.ps1 | base64 | tr -d '\n')
# 下发
ssh -i <key> -p 10022 -o StrictHostKeyChecking=no administrator@127.0.0.1 "powershell -NoProfile -EncodedCommand $B64"
```

三个好处：① 中文不丢；② 引号不炸；③ 整条命令纯字母数字，任何传输层不会弄脏它。
配套原则：**不解析 stdout**——脚本把结果写成文件（ndjson/结构化报告块），scp 拉回读。清理类脚本自己打印 `===== KILLED =====` 之类的报告块 + 复查块，方便判读。

### 4.1 远程读文件中文不乱码（只读巡检 Recipe）

**场景**：只读巡检——不能写文件、不能 scp，只能 `ssh ... "powershell ... Get-Content"` 把内容打回 stdout 看。

**坑（双重乱码）**：PowerShell 5.1 对**无 BOM 的 UTF-8 文件默认按 ANSI(GBK) 解码**，控制台再按 GBK 输出，经 ssh 拿回来就是双重乱码——实测 `package.json` 里的中文软件名变成了 `鐚庣考鏅鸿兘浣撹嚜鐢ㄧ増` 这种"锟斤拷"亲戚。

**标准做法（两道工序缺一不可）**：
```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8   # ① 控制台输出改 UTF-8
Get-Content "D:\下载\xxx.json" -Encoding UTF8             # ② 读文件显式按 UTF-8 解码
```

套进黄金范式整段照抄：
```bash
cat > /tmp/read_file.ps1 <<'EOF'
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
Get-Content "D:\下载\xxx.json" -Encoding UTF8
EOF
B64=$(iconv -f UTF-8 -t UTF-16LE /tmp/read_file.ps1 | base64 | tr -d '\n')
ssh -i <key> -p 10022 -o StrictHostKeyChecking=no administrator@127.0.0.1 "powershell -NoProfile -EncodedCommand $B64"
```

**判读**：中文正常 = 两道工序都对了；还是乱码 = ② 漏了 `-Encoding UTF8`（文件被按 GBK 读了）；中文全变 `????` = ① 漏了 `OutputEncoding`（输出端编码没切）。远程脚本首部固定写 `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8` 当肌肉记忆。

---

## 5. SSH 通之前的过渡：剪贴板中继模式

用户通过 UU远程盯着 Windows 屏幕，Agent 在 Mac 上 `printf '%s' '<命令>' | pbcopy`，用户到 Windows PowerShell 里 Ctrl+V 回车，再截图回报。规范动作：

- 每条消息只放**一条**命令，并口头声明"开头是 XXX"方便用户核对剪贴板内容（防"粘到上一条旧命令"，真踩过）。
- 命令设计成幂等/自愈：开头 `Remove-Item` 清旧文件、结尾输出标志性状态（`Get-Service sshd | Select Status,Name`）方便截图判读。
- 用户截图压缩后看不清细节时，引导用户把窗口文本直接复制粘贴回来当文本用。

---

## 6. Windows 现场排查命令速查

```powershell
reg query "HKLM\SOFTWARE\Microsoft\Cryptography" /v MachineGuid   # 机器指纹源（替身机器码算法用）
Get-FileHash 文件 -Algorithm SHA256                                # 哈希抽验
Get-Process | Where-Object {$_.Name -like "*runtime*"}            # 进程排查
netstat -ano | findstr "<PID>"                                    # 零外连观测
```

## 7. 远程操作四个高频坑（详见 pitfalls.md 远程操作类）

1. **SYSTEM 身份**：ssh 执行的命令跑在 SYSTEM 账户下，`$env:TEMP` 指向 `C:\Windows\Temp`——脚本一律写绝对路径。
2. **CLIXML + GBK 乱码**：远程输出机器解析不了——结果写成文件再 scp 拉回读；只读巡检不能写文件时，按 §4.1 双编码 Recipe（`OutputEncoding=UTF8` + `Get-Content -Encoding UTF8`）读。
3. **转义地狱**：嵌中文/引号必坏——EncodedCommand 黄金范式。
4. **Session 0 弹窗被吞**：GUI 出错对话框静默吞掉、进程"永久挂死"——探针加顶层异常兜底写日志 + 超时强杀；判断"活着"看 CPU/文件写入，不看窗口。
