# 踩坑全表（现象 → 根因 → 解法）

> 合并自 45 页实战教程（主）+ 上午虚拟机会话 + 晚场实体机会话（补充），共 40+ 条。
> 每条格式：**现象 → 根因 → 解法**，关键报错保留原文。按六大主题分组；文末附"虚拟机环境坑"（本机 VM 路线已被实战证伪，留档防走回头路）与"已否决路线"两张表。
> 编号沿用实战教程的踩坑实录编号（01–19 有跳号），便于对照原教程。

---

## 一、Electron / jsc 编译类

### 坑 12（探针 v1）：exe 跑探测脚本静默退出码 1 无输出
- **现象**：`ELECTRON_RUN_AS_NODE=1 runtime.exe probe.js`，静默退出，无任何输出。
- **根因**：exe 的 Fuses（Electron 出厂熔断开关）烧掉了 RunAsNode——环境变量说了不算。
- **解法**：改用 **app 目录模式**（`resources\app\package.json` + `main.js`），exe 无参数直接运行就会加载它。

### 坑 13（探针 v2）：exe 加载外部路径的脚本怎么摆都不对
- **现象**：让 exe 加载任意位置的脚本，各种摆法都失败。
- **根因**：Electron exe 启动时强制要求 `resources\` 目录存在且形态正确。
- **解法**：同上，app 目录模式，v3 起所有探针/编译脚本都用此模式，一轮通过。

### 坑 11'：裸 `require('bytenode')` 静默挂死
- **现象**：探针/编译脚本里裸 `require('bytenode')`，程序"永久挂死"无任何输出。
- **根因**：相对路径解析失败 → 报错对话框在 SYSTEM 会话被静默吞掉（见坑 11）。
- **解法**：bytenode 改**绝对路径** require + 顶层 `uncaughtException` 写盘后 `app.exit(1)`。同一个坑在全新场景复发——"症状→根因"档案值得反复复用。
- **适用边界**：**跨目录布局必须绝对**——探针/编译脚本在 `Temp\xxx-probe` 等目录、运行时 exe 和 bytenode 在另一处，相对解析必失败。例外：部署版入口 index.js 与 node_modules 同处 `resources\app\` 目录时，`require('bytenode')` 相对写法安全（require 从入口文件所在目录解析）。一句话：**和 node_modules 同目录的入口文件可相对，跨目录一律绝对。**

### 坑 04 / 环 6：跨架构/跨版本加载字节码必拒
- **现象**：Mac 上同版本 Node（ARM 版）加载目标 x64 字节码返回 `cachedDataRejected`；Windows ARM 虚拟机里 x64 的 runtime.exe 静默退出码 1，x64 PowerShell 退化成 x86 执行。
- **根因**：V8 字节码内含**架构 + 版本双重指纹**，跨架构必拒、同架构不同版本也拒。Apple 芯片的 QEMU/HVF/vz 不开放嵌套虚拟化，x64 程序只能靠软件模拟，Electron 起不来。
- **解法**：所有"加载、试、编译"动作都必须在 **x86-64 Windows + 同版本 V8** 环境做。M 系列 Mac 不要试图用 QEMU/UTM 跑 x64 Electron 实测，直接找实体机或商业 hypervisor。证据足够就换路线，"再试试"是赌博。

### 坑 16：凭字符串线索脑补接口
- **现象**：文档假设门禁函数 `gateActivation` 的参数是功能名（如 "extract"）。
- **根因**：凭字节码扒出的字符串"门禁×7 种功能"脑补（B 级线索 → C 级假设）。
- **解法**：挂钩日志一击修正——参数是应用安装路径（主程序写死的常量）。同时确认 `gateActivation` 是异步、返回结构就是 `{ok, error}` 无别字段。**教训：推断必须实测终审。字符串只能当线索（clue），不能当契约（contract）。**

### 经验：asar 里的 bytenode 包自身是可读源码（好消息）
- **现象**：解包后在 node_modules 里发现 bytenode 是明文 JS。
- **用法**：直接读它的导出能力（`compileFile` 等），不用猜工具能力边界。

---

## 二、asar 与指纹类

### 核心坑：重打 asar 首次启动报 `Integrity check failed for asar archive`
- **现象**：重打的 asar 自检全绿（6,362 文件一个不少），部署后首次启动exe 拒启，报 `Integrity check failed for asar archive`。
- **根因**：exe 的 Fuses 开着 ASAR 完整性校验——**打包时 asar 文件头的哈希"指纹"被嵌进 exe 资源**。换掉 asar 内容后，exe 里存的指纹对不上，软件拒绝启动。
- **解法**：不硬刚指纹，落地 **app 目录模式**（删掉 app.asar、以 `resources\app\` 目录形态部署）。校验只针对 asar 文件本身；没有 app.asar 时 Electron 走目录装载，fuse 无对象可校验，自然放行。全程不动 Fuses 开关、不篡改 exe。
- **注**：asar 重打工具只能打包 asar 文件，没有写回 exe 资源的工具；要换指纹得上 rcedit 级工具链，风险大，不值得。

### 坑 17：`unpackDir` 单元素花括号坑
- **现象**：`unpackDir: '{node_modules/better-sqlite3}'`（单元素花括号）自检 unpacked=0，原生模块被塞进 asar。
- **根因**：@electron/asar 3.2.8 匹配逻辑是 `dirPath.startsWith(pattern) || minimatch(dirPath, pattern)`；路径永远不会以带花括号字符串开头，且该版 minimatch 对单元素花括号不做展开。
- **解法**：`unpackDir: 'node_modules/better-sqlite3'`（去掉花括号，字面前缀匹配直接命中）。**教训：glob 模式在不同库版本间行为会漂移，能用字面前缀匹配就别用花括号。** better-sqlite3 的 .node 二进制必须保持 unpacked 外置，否则 Electron 无法加载。

### 坑 18：手工解 asar 按"JSON 头从偏移 16 开始"解出乱码
- **现象**：自写解析器按固定偏移读 asar 头，读到乱码。
- **根因**：新版 asar 头 8 字节对齐，老版 4 字节对齐；头部第一个字段就是 JSON 长度（上午场实战第一次也读到头 JSON 区域踩了同一个坑）。
- **解法**：别背偏移，**读字段**（动态读 JSON 长度再定位）。更省事的解法：`npx @electron/asar extract app.asar out-dir`，别用 7z（7z 不认识 asar 头，静默失败解不出内容）。

### 坑（补充）：@electron/asar 没有 `getRawHeader` API
- **现象**：重打包自检脚本调用 `getRawHeader` 不存在。
- **解法**：按 asar 文件格式手工读头（前 16 字节里 uint32LE@12 是头 JSON 大小，从偏移 16 读 JSON），自检三件套：`withIntegrity === files`、`unpacked` 计数与原包一致。
- **附带**：`unpackDir` 默认写到 `<dest>.unpacked`；若不出现则手动从基线复制未改动的 unpacked 内容兜底。

---

## 三、路径与数据类

### 坑 19：部署副本孤主进程挂死——版本错位
- **现象**：部署副本首次启动：孤主进程挂着、无渲染窗口、零输出；重定向 stdout 抓到 `cachedDataRejected`。
- **根因**：① 想当然从主程序字符串里的路径找基座，找到的其实是**用户数据目录**（缓存/试用记录/日志），真正程序树在开发目录的 release 产物里，主程序版本与预期不符；② 新版本的 Electron/V8 已升级，旧基座编译的字节码被拒收。
- **解法**：对照实验一锤定音——同一份 main.jsc 在旧版运行时加载 OK，问题在基座版本不在产物。再查用户数据目录最后修改时间，确认用户日常用哪个版本，**以该版本的完整程序树为基座重新部署**。教训：第 5 步"同版本编译"原则会反咬部署环节，**基座必须与编译时完全同款**。

### 数据目录写死：替身版会读写原版数据库
- **现象**：挂钩观察记录主程序启动时调 `setPath('userData', '<原厂写死路径>')`——与用户无关。
- **根因**：原厂把数据目录钉在固定位置；替身版照单全收就会踩"不动原数据"红线。
- **解法**：入口 index.js（自己的明文文件，先于主程序执行）做 **setPath 拦截器**：先预置独立目录，再拦截后续所有写入强制重定向。案例实测：原数据目录关键文件时间戳三次启动纹丝不动——拦截器生效铁证。

### 路径写法约定（远程文件操作）
- 向 Windows scp 过去的目标路径用 `C:/Users/...` **正斜杠**形式。
- Windows 侧解压优先 `Expand-Archive`（或 `tar -xf`），全 ASCII 路径最稳。
- 文件位置约定：**传文件放 `D:\下载`，装软件放 `D:\软件`**（安装包 InstallDir 也用 `D:\软件\<目标软件>自用版`，独立目录名避开原版）。

---

## 四、远程操作类（Mac 控 Windows）

### 坑 06：下载 OpenSSH 官方包的两连坑
- **现象**：`Invoke-WebRequest` 下载报完成、文件大小看似正常，解压报"找不到中央目录结尾记录"；curl 重下报 `CRYPT_E_REVOCATION_OFFLINE (0x80092013)`。
- **根因**：zip 文件被写坏（以实战教程为准）；国内网络够不到 CRL/OCSP 证书吊销服务器，curl 的 schannel 后端拒绝继续。
- **解法**：`curl.exe -L --ssl-no-revoke` 完整下载（`--ssl-no-revoke` 保留证书校验、只跳过吊销检查，比 `-k` 全放弃校验干净）。
- **补充记录（另一场实战）**：同体积文件也可能是**用户把同一条命令粘了两次、两个下载并发写同一个文件把 zip 写坏**——正确大小本来就是 5,704,583 字节。给用户的命令开头先 `Remove-Item` 删旧文件，并口头提醒"只粘一次"。

### 坑 10：命令行嵌中文/引号/`$`变量，层层转义必坏
- **现象**：经 ssh+cmd 多层转义后命令面目全非。
- **根因**：多层转义地狱。
- **解法**：**EncodedCommand 黄金范式**——写 .ps1 文件 → `iconv -f UTF-8 -t UTF-16LE script.ps1 | base64` → `powershell -NoProfile -EncodedCommand <base64>`，一个引号都不用转。好处：① 中文不丢（必须是 UTF-16LE，不是 UTF-8）② 引号不炸 ③ 纯字母数字任何传输层不会弄脏。详见 [remote-ops.md](remote-ops.md)。

### 坑 08：SSH 会话 `$env:TEMP` 指向 `C:\Windows\Temp`
- **现象**：脚本写的临时文件互相找不到。
- **根因**：通过 sshd 执行的命令跑在 SYSTEM 账户下，TEMP 是系统目录而非用户目录。
- **解法**：脚本里一律写**绝对路径**（如 `C:\Users\Administrator\AppData\Local\Temp`）。

### 坑 09：远程输出混着 XML 标记（CLIXML）和 GBK 乱码
- **现象**：机器解析不了远程输出；只读巡检时 `Get-Content` 读 UTF-8 文件，中文变双重乱码（如 `鐚庣考鏅鸿兘浣撹嚜鐢ㄧ増`）。
- **解法**：结果一律**写成文件再 scp 拉回读**，不解析 stdout。不能写文件的只读巡检，用 [remote-ops.md](remote-ops.md) §4.1 双编码 Recipe：脚本首部 `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8` + 读文件显式 `Get-Content -Encoding UTF8`，两道工序缺一不可。

### 坑 11：SYSTEM 会话（Session 0）GUI 出错被静默吞掉
- **现象**：探针程序"永久挂死"，stdout/stderr 全空。
- **根因**：Session 0 无交互窗口，报错对话框被静默吞掉。
- **解法**：探针程序加顶层异常兜底写日志 + 超时强杀；**判断"活着"看 CPU/文件写入/进程树，不看窗口**。

### 坑 14：探针 zip 用中文目录名
- **现象**：zip 内文件夹叫中文名，Windows `Expand-Archive` 报"非法字符"。
- **根因**：Mac 自带 zip 对中文文件名编码处理有问题，压出来 Windows 不认。
- **解法**：打包一律**全 ASCII 命名**，解压后在 Windows 侧重命名。

### 坑 15：整包重传慢
- **现象**：改一行 main.js 就重传 162MB 整包。
- **解法**：scp 传大 zip 约 1–2 分钟（走后台任务 + 完成通知，别前台干等）；**改 main.js 只需 scp 单文件（秒级）**，不要整包重传。

### 环 7：远控画面里看不到新建的文件夹
- **现象**：Windows 桌面明明建了文件夹，远控画面里没有。
- **根因**：远控画面同步延迟/权限。
- **解法**：补打单文件 zip + 交接文档 + 教用户三种找到文件的办法；别指望远控画面当文件管理器。

### 坑 H6 / 01 坑 10：中文输入法吞字符（远程键入的隐形杀手）
- **现象**：往 Windows 命令行/运行框注入的字符随机缺失、回车被吞。
- **根因**：中文输入法处于"中"模式时，注入的英文字符被输入法拦截。
- **解法**：固定 SOP——每次键入前先确认/切换到"英"模式，键入后截图验证行内容，回车无效时补回车。

### 01 坑 7：qemu hostfwd"假连通"
- **现象**：`nc -z 127.0.0.1 2222` 成功，但 SSH 握手永远完不成。
- **根因**：qemu 用户态网络转发在 guest 端口根本没监听时也会接受 TCP（只回 SYN），`nc -z` 只发 SYN 所以"通"。
- **解法**：探测 SSH 服务要看**协议横幅**而不是 TCP 连通：`curl -s telnet://127.0.0.1:2222` 或 bash `/dev/tcp` 读 banner（`SSH-2.0-OpenSSH...`）。注意 `/dev/tcp` 读法别用 `head -c 100`：OpenSSH 横幅仅 ~56 字节且服务器不主动断开，`head` 等不满会挂到超时（实测 ~1 分钟）；用 `head -c 40` 或外层包 `timeout 5`，几秒无输出即判"无 SSH 服务"。

### 01 坑 11：curl 在 Windows 上静默失败
- **现象**：cmd 里 curl 下完/信标没到，无任何报错。
- **解法**：别信退出码外观，用 `ping 网关` 分段诊断连通性；批处理里"失败也写标记文件"，让外面能区分"没跑"和"跑了但失败"。

### 02 补充：UU远程的两个坑
- **窗口被遮挡时键盘焦点传不到 Windows** → 先 activate_window 把远程窗口提到最前再打字。
- UU 的「终端」按钮实测只打开远程桌面而非终端 → 别指望它，正解是把 sshd 修好拿全功能通道。

### 02 补充：sshd 单连接调试模式限制
- `sshd -d` 是**单连接调试模式**（接待一个连接就退出）；`sshd -D` 是持久前台模式。要反复连接必须用 `-D` 或服务模式。
- 单连接限制下的 scp 变通：`ssh ... "cat > dest.zip" < local.zip`（stdin 走同一条会话）；或另开端口加映射。

### 02 补充：剪贴板中继模式（SSH 通之前的过渡）
- 用户通过远控盯着 Windows 屏幕，Mac 侧 `pbcopy` 放剪贴板，用户到 Windows PowerShell 里 Ctrl+V 回车，截图回报。
- 规范：每条消息只放**一条**命令，口头声明"开头是 XXX"防粘到上一条旧命令（真踩过）；命令设计成幂等/自愈（开头 `Remove-Item` 清旧文件、结尾输出标志性状态方便截图判读）。

---

## 五、Windows 服务类

### sshd 修复完整链（事件 7000/7009，本 Skill 远程作战的第一仗）

**诊断三件套**：
1. `sshd -V` 做二进制 sanity check（活没活着）；`sshd -d` 静默秒退无输出 = 连初始化都做不到。
2. 事件日志：`Get-WinEvent -FilterHashtable @{LogName="System"; ProviderName="Service Control Manager"} -MaxEvents 6`——反复报 **7000（服务启动失败）/ 7009（30 秒连接超时）**。
3. 案例实测：`sshd.exe -d` 返回 `exitcode=-1073741511`（0xC0000139 = STATUS_ENTRYPOINT_NOT_FOUND，依赖 DLL 入口点缺失，典型的"系统优化软件搞坏系统组件"混合二进制）；二进制日期停在 2019 年旧版。

**失败方案（按尝试顺序，别再走）**：
1. `Remove-WindowsCapability + Add-WindowsCapability` 重装：拉回**同一份坏文件**，无效（坑 05）。
2. `Invoke-WebRequest` 下载官方包：文件被写坏解压失败（见坑 06）。
3. `sc.exe config binpath` 指向新二进制：服务**快速失败**——根因不是路径，是宿主密钥权限。

**成功方案（两步，照抄）**：
```powershell
# 第 1 步：curl 下载（跳过证书吊销检查），开头先删旧文件
Remove-Item "$env:USERPROFILE\Downloads\ossh.zip" -Force -ErrorAction SilentlyContinue
curl.exe -L --ssl-no-revoke --retry 3 -o "$env:USERPROFILE\Downloads\ossh.zip" "https://github.com/PowerShell/Win32-OpenSSH/releases/latest/download/OpenSSH-Win64.zip"
# 第 2 步：解压 + 删旧服务 + 官方脚本重建
$D="C:\Program Files\OpenSSH"; Expand-Archive -Force "$env:USERPROFILE\Downloads\ossh.zip" $D
sc.exe stop sshd | Out-Null; sc.exe delete sshd
powershell -NoProfile -ExecutionPolicy Bypass -File "$D\OpenSSH-Win64\install-sshd.ps1"
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic    # ★ 设自启动，Windows 重启后通道不能断
```
**验收两条缺一不可**：`Get-Service sshd` 显示 `Status: Running` **且** `StartType: Automatic`——只 Running 不算过（`Manual` 意味着重启后通道即断，本 Skill 把通道当长期资产用）。
**根因**：新 sshd 前台 `-d` 跑得欢、服务模式却失败——host key 私钥 ACL 违反 **StrictModes**（Administrator 账户对 `C:\ProgramData\ssh\ssh_host_*_key` 有访问权，服务以 LocalSystem 运行时被拒）。`install-sshd.ps1` 的 FixHostKeyPermissions 修的正是这件事。**以后遇到"前台行、服务不行"的 sshd，直接跑 install-sshd.ps1，别手工折腾 ACL。**

### Windows 上安装 OpenSSH Server 的最小命令链（干净系统用）
```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Set-Service -Name sshd -StartupType Automatic
Start-Service sshd
New-NetFirewallRule -DisplayName 'OpenSSH...'   # 放行 22 端口
```
注意：`Add-WindowsCapability -Online` **需要联网下载**——在无人值守安装的网络未就绪阶段会静默失败，应挪到首登阶段执行。

---

## 六、验证类

### "找不到 app-update.yml 已放弃"是好消息
- 日志里出现这行 = 自动更新已断（更新器找文件失败一次后放弃，启动不受影响），正是我们部署时删 `app-update.yml` 想要的效果。

### SSH 是 SYSTEM 身份，看不到界面
- 远程会话跑在 SYSTEM 账户下，GUI 什么样验证不了。**视觉确认（无激活弹窗、界面显示自用版）明确留给用户本人**——验证报告分【已机器验证】与【需人工目验】两栏，不夸大。

### 静默安装 `/S` 跳过快捷方式 ≠ 缺陷
- 常见行为。交付时给用户两个口径：界面安装，或右键"发送到 → 桌面快捷方式"。

### 安装版与便携目录版共用同一用户数据目录
- 同名单实例锁，同时开两个会互踢——属设计预期，写进交付说明即可。

### 安装包哈希抽验
- Windows 侧 `Get-FileHash` 与 Mac 侧 `shasum` 对比前 16 位必须一致——证明传输链路没弄脏安装包。移动入基线目录后**再校验一次**。

### "断网混过试用期"对原厂不可行、对替身无意义
- 探针实测：试用暂停判定在作者服务器，断网启动云端一回"试用已暂停"照样拦（这也是已否决路线之一）。替身零联网设计，天然免疫断网场景。

---

## 附 1：虚拟机环境坑（本机 VM 路线已被实战证伪，留档防走回头路）

> 结论先行（以实战教程为准）：Apple Silicon Mac 上，QEMU/HVF、macOS vz 框架、UTM 套壳**全部不开放嵌套虚拟化**，Windows ARM 客户机的 x64 模拟起不来，Electron 应用跑不了。**不要走本机虚拟化，直接实体机 + 远程通道，或商业 hypervisor（Parallels/VMware Fusion）。** 以下坑表留档只为防回头。

| # | 现象 | 根因 | 解法 |
|---|---|---|---|
| 01–03 | QEMU+HVF / vz / UTM：x64 PowerShell 退化成 x86、runtime 静默退出码 1、ffmpeg.dll 加载报错 | 苹果芯片不开放嵌套虚拟化，换皮不换代 | 放弃本机虚拟化 |
| H1 | autounattend.xml 装机在文件复制 100% 后整体校验被拒 | 某段元素排列顺序不合微软规范（前半段不校验，后半段整体拒） | 按规范重排；或整个去掉 specialize 阶段，Shell-Setup 子元素顺序固定为 `AutoLogon → OOBE → UserAccounts → FirstLogonCommands` |
| H2 | 修复后重装仍失败（"幽灵旧系统"） | 没格盘直接重启，旧系统残留在虚拟盘里抢引导 | 重建虚拟盘 + 重置固件变量 |
| H3 | chntpw 参数吃不下（只打印帮助） | 工具不兼容 | 转用 hivex（Alpine 里 Perl 模块叫 Win::Hivex，与文档常见名不同，import 报错即此因） |
| H4 | 登录页一直报"密码错误" | 新式 Windows 账户禁用标志藏在 SAM 文件 F 值偏移 **0x38**（0x30 是另一字段，极易认错；另一份实战记录为 0x56——两处都以"F 值内偏移"定位，以实战教程为准） | 0x211（禁用）→ 0x214（启用+允许空密码），经"导出哈希→字节级比对"取证后落刀 |
| H5 | 首登批处理"跑了但秒退" | cmd.exe 解析 LF 行尾批处理把两行拼成一行；中文注释在切换代码页前是乱码 | 规矩立死：批处理必须 **CRLF 行尾 + 纯 ASCII**，chcp 65001 放第一行 |
| H7 | virtio 网卡驱动装不上 | ARM64 客户机无驱动 | 换 e1000 网卡（Windows 自带驱动）——求稳优先用 guest 自带驱动的虚拟硬件 |
| H8 | e1000 时通时断（"PING: 传输失败"），curl 静默失败 | ARM64 客户机兼容性问题 | 放弃"网络看守"，全部改走共享文件盘 |
| H9 | FAT32 共享盘在 Windows 关机途中被 Mac 抢挂，文件系统标脏 → Mac 侧只读 | 跨系统挂载冲突 | 不修（修复要管理员），直接重建新镜像（2 分钟比修复快） |
| H10 | QEMU 重启报 `Property 'virtio-net-device.netdev' can't take value 'n0', it's in use` | 两块虚拟网卡共用一个 netdev | 删 virtio 网卡只留 e1000；排错看 daemonize 后的 qemu.log |
| 环 2 | 看守脚本报"SSH READY"但连接挂起 120 秒零输出 | 端口通 ≠ 服务可用（见 01 坑 7） | 规程：看守任务检测协议 banner，不检测空端口 |
| 环 3 | 虚拟显卡无 GPU 加速，显示异常、鼠标点击从未生效 | 虚拟显卡限制 | Alpine 救援盘直接引导，挂串口当控制台，挂 ntfs3 读虚拟硬盘真实日志（`modprobe ntfs3 && mount -t ntfs3 -o force /dev/nvme0n1p3 /mnt/win`，读 `$Windows.~BT/Sources/Panther/setuperr.log`——注意 `$Windows.~BT` 必须单引号禁变量展开） |
| 01 坑 5 | OOBE（开箱设置）在虚拟显卡上黑屏卡死，反复失败 | 系统挂在"等待人工点下一步"的黑屏；注册表残留自动触发"重新 specialize"的命令 | Alpine 救援盘 + hivex 离线改注册表：跳过 OOBE（挂起参数清零）+ 预设自动登录 + 布置首登脚本 |

## 附 2：已否决的路线（别走回头路，以实战教程为准）

| 路线 | 否决理由 |
|---|---|
| 破解激活算法 | 对抗 RSA 非对称加密数学上不可行；等同伪造签名，越红线 |
| 爆破/内存补丁 | 每次启动都要补，易碎；改原厂核心代码，越红线 |
| 劫持作者服务器（改 hosts 指假服务器） | 要实现整套协议；仍是欺骗性质 |
| 断网混过试用期 | 试用暂停判定在作者服务器，一启动云端就回"试用已暂停"，此路堵死 |
| 伪造激活码 | 授权模块内嵌 RSA PUBLIC_KEY，激活码是带签名的编码 JSON，造不了假码 |
| 伪造激活响应/签名 | 包里是 RSA 公钥验签，无私钥，硬碰必败 |
| 直接改 exe 的 Fuses 当 Node 工具用 | 把 exe 当普通 Node 工具会误判，且动原厂文件 |
| 关 ASAR 完整性校验来重打包 | 不以关闭校验代替正确打包；正确姿势是 app 目录模式，fuse 一个没动 |

## 附 3：流程级教训（不属技术坑，但每个都烧过真金白银）

| 坑 | 教训 |
|---|---|
| 不设截止时间的死磕 | "把每条坑都填平"而不是及早换路线，5 小时没碰到任务本体。**给路线设时间盒，到点果断切换；备选方案提前想好挂在墙上** |
| 用户给错选项编号 | 用户回了不存在的选项时先澄清再执行。给选项编号醒目、一次不超过 3 个、给推荐 |
| 后台看守任务通知打断当前回合 | 区分"旧看守的超时通知"和"新事件" |
| fork 型子智能体 ≠ 并行劳动力 | 要并行就派明确分工的独立任务；同一条流水线用 resume 续派同一 agent |
