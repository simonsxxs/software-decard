# software-decard Skill 白纸测试报告

- 测试日期：2026-10-12
- 测试模式：白纸测试员（只读 Skill 文档 → 真实环境照做 → 带回真实输出）
- 测试范围：Skill 允许的只读子集（Windows 侧全程零写入、零进程操作；Mac 侧仅写本报告）
- 测试人：subagent（白纸测试员）

---

## 一、环境

| 项 | 实测值 |
|---|---|
| Mac → Windows 通道 | `ssh -i probe_key -p 10022 administrator@127.0.0.1`（UU远程端口映射） |
| SSH 横幅 | `SSH-2.0-OpenSSH_for_Windows_10.0 Win32-OpenSSH-GitHub` |
| 登录身份 | `testbed-pc\administrator`（主机名 TESTBED-PC） |
| PowerShell 版本 | 5.1.19041.6093 |
| sshd 服务 | **Running**，启动类型 **Manual**（见缺陷 D6） |
| 目标部署目录 | `D:\软件\target-local`（存在） |
| 远程编码基线 | 脚本首部固定 `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8`（见缺陷 D2） |

连通性：一次通过，无需重试。`nc -z` 通、横幅真、ssh 登录真、EncodedCommand 真。

---

## 二、步骤 1：连通性 + 远程范式测试（✅ 通过）

**① 端口与横幅（remote-ops.md §3 验证三连）**

```
$ nc -z -w 3 127.0.0.1 10022 && echo PORT_OPEN
PORT_OPEN
$ (exec 3<>/dev/tcp/127.0.0.1/10022 && head -c 60 <&3 | cat -v)
SSH-2.0-OpenSSH_for_Windows_10.0 Win32-OpenSSH-GitHub^M
```
注：文档原文 `head -c 100` 阻塞近 60 秒才返回（banner 仅 ~56 字节且连接不断开，head 等不满 100 字节）。见缺陷 D1。

**② 登录实测**
```
$ ssh -i ... -p 10022 ... administrator@127.0.0.1 "whoami & hostname"
testbed-pc\administrator
TESTBED-PC
```

**③ EncodedCommand 黄金范式（remote-ops.md §4 照抄）**

实际命令（范式不变，因 Mac 只读约束把"写 .ps1"换成 printf 管道，编码链路完全一致）：
```bash
B64=$(printf '%s' "$PS1_SCRIPT" | iconv -f UTF-8 -t UTF-16LE | base64 | tr -d '\n')
ssh -i <key> -p 10022 ... administrator@127.0.0.1 "powershell -NoProfile -EncodedCommand $B64"
```

真实输出：
```
5.1.19041.6093
#< CLIXML
 Status Name StartType
 ------ ---- ---------
Running sshd    Manual
2026-10-12 01:31:57
```
中文/引号/`$` 变量在脚本内任意写，经 base64 后零转义——**范式照抄可用，中文不丢、引号不炸**。CLIXML 尾部垃圾与文档坑 09 描述一致。

---

## 三、步骤 2：静态侦察验证（✅ 全部命中）

检查清单逐项（SKILL.md §4 第 1 步"摸清骨架"的已部署版对照）：

**1) package.json 关键字段** — `D:\软件\target-local\resources\app\package.json`
```json
{ "name": "target-local", "productName": "目标软件自用版",
  "version": "1.2.1-local.1", "edition": "target-local",
  "main": "index.js",
  "dependencies": { "bytenode": "^1.5.3", "better-sqlite3": "^12.11.1", "electron-updater": "^6.8.9", ... } }
```
main 指向明文入口 index.js，bytenode 在依赖里——与 Skill 架构一致。

**2) index.js 的 userData 拦截（setPath）** — 全文 1010 字节，结构与 SKILL.md 第 6 步"关键代码"同构：
```js
const LOCAL_USERDATA = path.join(app.getPath('appData'), 'target-local');
app.setPath('userData', LOCAL_USERDATA);
const origSetPath = app.setPath.bind(app);
app.setPath = function (name, p) {
  if (String(name).toLowerCase() === 'userdata') p = LOCAL_USERDATA;   // 拦截器
  return origSetPath(name, p);
};
require('bytenode');
require('./main.jsc');
```
双重隔离（预置 + 拦截）逐字落地。注释还自证了动机："1.2.1 的 userData 写死在 D:\原厂目录\目标软件，防串扰"。

**3) 授权替身模块文件存在** — `resources\app\core\` 下：
```
activation.js   5,495 字节   （明文替身源码）
activation.jsc  7,080 字节   （字节码替身）
```
**activation.jsc = 7,080 字节，与 SKILL.md 第 5 步"案例实测值 7,080 字节"分毫不差**（验证矩阵第 6 项"替身确认真身"的直接证据）。导出名单（13 个成员）与 prompts.md 附录③ 逐字一致：
```js
module.exports = { getMachineId, inviteCodeFor, verifyActivation, loadActivation,
  saveActivation, isActivated, getLicenseState, submitActivation, claimCard,
  gateActivation, setOnlineUrl, TRIAL_DAYS, LICENSE_LABELS };
```

**4) jsc 字节码文件** — 全目录仅 2 个：`main.jsc`（167,872 字节，原厂主程序）+ `activation.jsc`（替身）。其余 core/ 全部明文 JS。

**5) app 目录模式部署特征** — 全部命中：
```
顶层：目标软件自用版.exe (188,784,128) + ffmpeg.dll/libGLESv2.dll/...  （Electron 运行时树，exe 已改名）
resources\：app\ 目录 + app.asar.unpacked\ + elevate.exe
app.asar        → False（已移除）
app-update.yml  → False（更新渠道已断）
```
另：`app.asar.unpacked\node_modules\better-sqlite3\build\Release\better_sqlite3.node`（1,918,976 字节）在位——原生模块 unpacked 外置，符合 pitfalls.md 坑 17 的要求。

---

## 四、步骤 3：验证矩阵只读子集

| 矩阵项 | 结果 | 证据（节选） |
|---|---|---|
| #3 原厂数据零触碰 | ✅（只读限度内） | 原厂 `D:\原厂目录\目标软件` 最后写入 **2026/10/11 21:48:38**（trial.json 21:48:36）；隔离目录 `C:\Users\Administrator\AppData\Roaming\target-local` 持续写入至 **23:50:57**。切换后原厂目录零新写入，隔离目录独立演化。 |
| #4 零外连/无监听 | ✅（进程未运行态） | `Get-Process` 无任何 target-local/目标软件进程 → 无对外连接、无监听端口。全机 LISTENING 仅 `0.0.0.0:22`(sshd, PID 1428) 与 `0.0.0.0:80`(System, PID 4，系统 HTTP.sys，与本软件无关)。 |
| sshd 状态 | ✅ | `Running`（见缺陷 D6：StartType=Manual，重启后不自启）。 |

如实说明：矩阵 #1/#2/#5（冷启动、日志、重启稳定性）需启动/杀进程，超出本次只读授权，未执行；#4 的运行态 netstat 同理。隔离目录内 `logs\main.log`（1,184 字节）存在，内容未读（属读取用户数据边缘，保守跳过）。

---

## 五、步骤 5：D:\下载 安装包排查

**官方原版安装包（~113MB Setup）：未找到。** D:\下载 中 >50MB 文件 22 个，无原版目标软件安装包。

**找到的是本 Skill 的前次交付物（反向印证 Skill 被执行成功过）：**

| 文件 | 大小 | 备注 |
|---|---|---|
| `target-local-win.zip` | 212,586,909 B（≈212MB） | 与 SKILL.md 第 9 步"绿色版 zip 案例 212MB"吻合 |
| `目标软件自用版-Setup-1.2.1-local.1.exe` | 159,481,966 B（≈159MB） | 与"NSIS 安装包案例 159MB"吻合 |

对 Setup 的只读验证：
- NSIS 指纹：文件尾 2048 字节含 `Nullsoft` 标记 → **True**（确是 NSIS 包）；
- 版本信息：ProductName/Version 全空（makensis 默认未写版本资源，非缺陷但交付时可加强）；
- SHA256（Windows 侧）：`7BF9116EF20306B0CE0CC4444C904E4E08F63845198779C396D731BF963C6FF9`（Mac 侧无此文件，矩阵第 8 项双端对比无法闭环）；
- 7z 可用性：**7z NOT FOUND**（见缺陷 D3）。

**若只有安装包该怎么入手（按 SKILL.md 第 1 步叙述 + 只读可验证部分）**：
1. `shasum -a 256` 校验哈希，移入基线目录再校验一次（哈希抽验防传输弄脏）；
2. 解包四连拆：`7zz x Setup.exe -obaseline/installer` → 取 `$PLUGINSDIR\app-64.7z` → 解开程序树 → `npx @electron/asar extract app.asar out-dir`（7z 解不了 asar）；
3. 逐文件 SHA-256 清单做基线，盘点明文 JS / .jsc 字节码 / node_modules 三类；
4. `strings main.jsc | grep -E 'gateActivation|getLicenseState|license'` 只出假设；
5. 记录自动更新渠道与 Fuses 状态两个隐患，再决定探针方案。
本次只读约束下可验证到第 1、5 步的探测性动作（NSIS 指纹 sniff、哈希、7z 可用性均已做），解压与 asar 拆解属写操作，未执行。

---

## 六、缺陷清单（本测试最重要产出）

### 文档缺陷（照抄/照做时的真实摩擦）

**D1（轻微）remote-ops.md §3 验证三连 ② 会假死约 1 分钟**
- 位置：references/remote-ops.md §3「`(exec 3<>/dev/tcp/127.0.0.1/10022 && head -c 100 <&3 | cat -v)`」
- 现象：OpenSSH 横幅仅 ~56 字节，服务器不主动断开，`head -c 100` 永远等不满，直到对端闲置超时（实测阻塞 ~55–60 秒才返回）。在交互式执行中极易被误判为"映射假连通"去重试。
- 绕过：改用 `head -c 40`，或外层包 `timeout 5`。

**D2（中等）远程读 UTF-8 文件双编码坑无任何Recipe**
- 位置：references/remote-ops.md §4 / pitfalls.md 坑 09（只说了"写成文件再 scp 拉回"，未给交互式只读的等价做法）
- 现象：只读巡检场景（不能写文件）下，`Get-Content xxx.json` 经 ssh 拿回来是双重乱码——PS 5.1 对无 BOM 的 UTF-8 文件按 ANSI(GBK) 解码，再经 GBK 控制台输出。实测 package.json 的 `目标软件自用版` 变成 `鐚庣考鏅鸿兘浣撹嚜鐢ㄧ増`。
- 绕过：脚本首部加 `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8`，且读文件必须显式 `Get-Content -Encoding UTF8`。两者缺一不可，文档未提。

**D3（中等）第 1 步关键命令假设目标机装有 7z，实际没有**
- 位置：SKILL.md §4 第 1 步「关键命令（原文照抄）」`7zz x ...`
- 现象：目标 Windows 实测 `7z NOT FOUND / 7zz NOT FOUND`。命令照抄即失败，Skill 通篇只给了 OpenSSH 与 makensis 的获取/安装链，没有 7-Zip 的安装指引。
- 绕过：安装包可改在 Mac 侧用 7z/keka 解；zip 类可用 `Expand-Archive`。建议 Skill 补一条"目标机无 7z 时的等价路径"。

**D4（轻微）relative vs absolute 的 bytenode require 规则没说清适用边界**
- 位置：SKILL.md 第 6 步示例 `require('bytenode')`（相对）vs prompts.md 附录① / pitfalls.md 坑 11' 要求"bytenode 改绝对路径 require"
- 现象：两套写法并存，文档未说明何时相对路径安全（入口 index.js 与 node_modules 同处 app 目录时安全，实测部署目录即用相对且正常）、何时必须绝对（探针/sandbox 跨目录布局）。
- 绕过：按部署/探针语境自行判断。建议各加一句适用条件。

**D5（轻微）sshd 修复成功标准漏了"自启动"**
- 位置：references/remote-ops.md §2.3「通过标准：Status: Running」
- 现象：本机 sshd `Running` 但 `StartType = Manual`——Windows 重启后通道即断，而 Skill 把通道当长期资产用。成功标准未包含 `Set-Service sshd -StartupType Automatic`。
- 绕过：本次无需处理；建议文档把自启动写进验收。

**D6（轻微）文件位置约定与第 1 步命令的工作目录前提脱节**
- 位置：SKILL.md §4 第 1 步命令用相对路径 `baseline/installer`、`app.asar`；remote-ops.md §3 又立规"传文件放 D:\下载"
- 现象：第 1 步命令没说明在 Mac 工程目录执行还是在 Windows D:\下载 执行，两套路径约定并存，照抄者要自己拼接。
- 绕过：按 Mac 侧工程目录理解后执行无碍。建议第 1 步开头写明执行侧与 cd 前提。

### 环境缺陷（非 Skill 之过，如实记录）

**E1** 官方原版安装包（~113MB）不在 D:\下载，步骤 5 的"只有安装包"实战无法闭环；找到的 159MB Setup 与 212MB zip 均为本 Skill 前次交付物（这反而构成 Skill 有效性的实物证据）。
**E2** 只读约束下无法启动软件，验证矩阵 #1/#2/#4 运行态/#5 未执行；#3/#4（静态部分）/sshd 已完成。

---

## 七、结论

**Skill 可以照做。** 在零经验、只读 Skill 文档的前提下：
- 远程通道部分（remote-ops.md 验证三连 + EncodedCommand 黄金范式）照抄即通，一次成功；
- 静态侦察检查清单对真实部署目录 `D:\软件\target-local` 逐项命中：app 目录模式（无 app.asar / 无 app-update.yml / resources\app 在位）、setPath 双重拦截器、activation.js+jsc 替身（**7080 字节与文档案例值精确一致**）、main.jsc 原厂字节码、原生模块 unpacked 外置；
- 验证矩阵只读子集（原厂数据零触碰、零外连静态证据、sshd 状态）全部通过，且隔离目录与原厂目录的时间戳演化与"拦截器生效"叙事吻合。

**需修后再发布**：D2（UTF-8 读取 Recipe）、D3（7z 缺失的等价路径）、D1（横幅检查防假死）、D5（sshd 自启动验收）。均为文档级修补，不涉及方法论错误。核心工程论断（app 目录模式绕过 ASAR 指纹、替身契约、V8 同版编译）在本机实物上均找到对应证据，未发现任何与事实相悖的叙述。

**未覆盖**（只读授权之外）：探针/挂钩/编译回环（需写文件与启动运行时）、12 项矩阵中的 5 项运行态、安装包哈希双端对比（Mac 侧无该文件）。

---

## 修复记录（2026-10-12）

- **D1**（横幅检查假死）→ references/remote-ops.md §3「Mac 侧验证三连」：② 改为 `timeout 5 bash -c '... head -c 40 ...'`，补"横幅仅 ~56 字节且服务器不主动断开，`head -c 100` 会挂 55–60 秒"的防假死说明与判读（几秒内出 `SSH-2.0-` = 真 SSH；timeout 杀出 = 回去修 sshd）；references/pitfalls.md「01 坑 7」同步加同一句。
- **D2**（UTF-8 远程读取双编码无 Recipe）→ references/remote-ops.md 新增 §4.1「远程读文件中文不乱码（只读巡检 Recipe）」：`[Console]::OutputEncoding=[System.Text.Encoding]::UTF8` + `Get-Content -Encoding UTF8` 双保险，含 EncodedCommand 版整段照抄命令与乱码判读；references/pitfalls.md 坑 09 补双重乱码现象并指向该 Recipe。
- **D3**（7z 缺失）→ SKILL.md §4 第 1 步「关键命令」后新增"机器上没有 7z 时的等价路径"：Mac 侧 `brew install sevenzip`（推荐）、Windows 侧 `winget install --id 7zip.7zip -e`（命令名换 `7z`）、zip 类用自带 `Expand-Archive`（NSIS 内核是 7z 解不了仍须走前两条）。
- **D4**（bytenode 相对/绝对 require 边界）→ SKILL.md 第 6 步、references/prompts.md 附录①、references/pitfalls.md 坑 11' 三处各补适用边界一句：与 node_modules 同处 `resources\app\` 的部署版入口 index.js 相对 require 安全；探针/编译脚本跨目录布局必须绝对路径。
- **D5**（sshd 漏自启动验收）→ references/remote-ops.md §2.3 与 references/pitfalls.md §五「成功方案」：命令链补 `Set-Service -Name sshd -StartupType Automatic`；通过标准改为双验收——`Status: Running` 且 `StartType: Automatic`，注明"只 Running 不算过，Manual = 重启后通道即断"。
- **D6**（工作目录前提不明）→ SKILL.md §4 第 1 步「关键命令」开头写明"在 Mac 侧工程目录执行，先 cd 进工程目录"，并在命令行内标注 cd 时机（strings 前 `cd out-dir`、find 前 `cd ..`），同时说明与 remote-ops.md「文件位置约定（D:\下载）」的分工；references/remote-ops.md §2 开头补"本节在 Windows 管理员 PowerShell 执行、全绝对路径、与工作目录无关"。

自检：本报告发布版已对目标软件标识与测试主机名做机械脱敏，其余为测试员原文照抄。
