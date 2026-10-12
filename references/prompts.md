# 分步 Agent 提示词全套（15.0–15.6）+ 代码模板附录

> 用法：改造目标软件时，把【软件名】【模块名】【通道名】替换成实际值，整段粘贴给任何 AI 编程助手即可。
> 15.0 是项目开场总控；15.1–15.6 依次对应探针 → 挂钩 → 对账 → 写替身 → 编译回环 → 部署验证六步。
> 本文末附录是每段提示词配套的完整代码模板（替换契约值与路径即用）。

---

## 15.0 总控提示词（项目开场用）

- **用途**：项目开场，确立红线、工作方式和第一阶段任务。
- **使用时机**：刚接手一个"去卡密自用版"改造任务，第一阶段只做只读侦察。

```text
你是一名资深逆向工程助手。目标：把我已购买授权的【软件名】改造为免卡密自用版。
铁律：① 不伪造激活签名、不请求作者服务器、不改原厂字节码文件；
② 一切改动集中在我方新增/明文文件中，可随时回滚；
③ 接口契约只信运行实测，字符串线索只当假设；
④ 每阶段结束把结论与证据写入工程文档（HANDOFF.md / design.md）。
先只做只读侦察：解包安装包，盘点技术栈（运行时与版本、明文/编译文件分布、
更新渠道、用户数据位置），输出清单与未知项列表，等我批准后再进入下一阶段。
```

---

## 15.1 探针制作提示词（第 2 步·动态探测接口契约用）

- **用途**：让软件自己的运行时把授权模块的接口"问出来"（导出名单、每个函数的同步/异步/抛错形态）。
- **使用时机**：静态侦察发现授权模块是字节码（.jsc）读不了，需要动态实测契约。

```text
目标软件的授权模块是 bytenode 字节码（.jsc）。制作一个"探针包"，要求：
1. 用软件自带的运行时 exe，以"应用目录模式"（resources\app\package.json +
   main.js）启动探针逻辑，禁止依赖 ELECTRON_RUN_AS_NODE（Fuses 已禁用）；
2. main.js 里：require bytenode（绝对路径）→ require 授权模块（绝对路径）
   → 枚举导出 → 逐个函数安全调用（记录同步返回 / 异步落定 / 抛错三种形态）
   → 结果写 ndjson 日志（每条含时间戳与函数名）；
3. 用户数据目录重定向到临时位置，绝不读写真实用户数据；
4. 顶层 uncaughtException 兜底：异常写盘后主动 app.exit(1)；
5. 打包 zip 全部用 ASCII 命名；附一份给真人看的"双击运行"bat 与说明。
输出：探针包 + 设计说明（哪些地方可能踩坑）。
```

**配套用户指令（双击即用版，发给真人执行时）**：

```text
目标软件是 Electron 应用，授权模块 activation.jsc 是 bytenode 字节码。做一个"探针包"：
用软件自带的运行时以应用目录模式加载授权模块，逐个导出调用并记录返回结构（区分同步/异步），
输出 ndjson 结果。要求：① 全 ASCII 命名可 zip 传输；② 用户数据目录重定向到临时位置，绝不写真
实数据；③ 顶层异常兜底写日志；④ 双击即用，不需要用户装任何东西。
```

---

## 15.2 挂钩观察提示词（第 3 步·观察原厂真实调用用）

- **用途**：给授权模块装上"窃听器"，记录主程序实际调用（函数、顺序、真实参数、返回值去向）。
- **使用时机**：探针给出了"能问出什么"，挂钩要确认"真对话是什么"——两者互证后契约定案。

```text
做一个"观察副本"：复制整套 app 目录，把入口 index.js 换成挂钩版——拦截
Module._load，当加载目标为授权模块（activation.jsc）时，用代理包装其每个导出：
调用前记录函数名与参数，返回后记录结果（异步函数等 Promise 落定再记）。
日志追加写 ndjson。然后正常运行原厂主程序 10 秒以上，覆盖启动流程。
重点回答：① 主程序实际调了哪些函数、按什么顺序；② 每个函数的真实参数
（与探针假设是否一致）；③ 返回值被如何使用；④ 哪些导出从未被调用。
```

---

## 15.3 消费方对账提示词（第 3 步·渲染层源码契约核对用）

- **用途**：读界面明文代码，字段级核对授权 IPC 通道的期望返回结构，产出"消费契约表"。
- **使用时机**：Electron 软件的界面（renderer）和 preload 桥通常是明文 JS，是契约的 A 级证据源。

```text
软件是 Electron 应用，渲染层（renderer）和 preload 桥是明文 JS。通读全部明文
JS 中调用授权相关 IPC 通道的代码，输出一张"消费契约表"：每个通道名 → 调用参数
→ 期望返回的每个字段 → 每个字段在什么 UI 分支被消费（含文件名:行号）。
特别注意：① 成功/失败/异常分支分别消费哪些字段；② 有没有字面量比较
（如与 'permanent' 比较）；③ 日期字段的解析方式；④ 哪些通道定义了但无调用点；
⑤ 多个窗口是否共用 preload。明确区分【源码实证】与【推断】。
```

---

## 15.4 替身模块编写提示词（第 4 步·编写 activation.js 用）

- **用途**：按契约表写一个诚实的替身授权模块。
- **使用时机**：探针（S 级）+ 挂钩（S 级）+ 源码对账（A 级）三源互证、契约定案之后。

```text
按附件"消费契约表 + 导出清单"编写替代授权模块（普通 CommonJS，可编译为字节码）：
1. 导出成员与原厂清单逐一对应，一个不缺；
2. 所有函数容忍任意参数（内部忽略多余参数）且永不抛异常；
3. getLicenseState 返回：已激活 + licenseType 用源码里被字面量比较的档位值
   + licenseLabel 用"自用版" + expire 为空串；
4. 门禁函数返回 Promise.resolve({ok:true})；
5. 激活码/邀请码相关函数诚实返回"自用版无需激活码/不适用"，不伪造成功；
6. 机器码优先复刻原厂算法（读注册表 MachineGuid 的 md5），读不到再兜底；
7. 零联网、零试用文件、零注册表卡密项；内部异常写本地日志（限 100KB 轮转）。
写完先在本机用 node 跑一遍导出对账（Object.keys 与清单比对）再交付。
```

---

## 15.5 编译与回环验证提示词（第 5 步·jsc 编译用）

- **用途**：在同款运行时里把替身编译成 .jsc 并立即回环自验。
- **使用时机**：替身模块编写完成、导出对账通过后。**必须在目标软件的同款 Electron 运行时里做**（V8 版本必须一致）。

```text
在目标软件的同款运行时里编译替身模块为 .jsc 并回环验证：
1. 编译脚本以应用目录模式跑在目标 runtime 里，bytenode 用绝对路径 require；
2. bytenode.compileFile 编译后同进程立即 require 产物；
3. 按契约清单逐项体检：导出数量与名字 / getLicenseState 全字段 / 门禁是否
   Promise 且落定 {ok:true} / 各拒绝函数返回文案 / TRIAL_DAYS 与 LICENSE_LABELS；
4. 结果写 ndjson 后退出（app.exit），不解析 stdout；
5. 若产物加载报 cachedDataRejected，立即报告运行时版本差异，不要重试。
```

---

## 15.6 部署验证提示词（第 7–8 步·部署与证据化验证用）

- **用途**：以 app 目录模式部署替身版，做证据化验证（机器验证与人工目验分列）。
- **使用时机**：编译回环通过后。

```text
部署替身版到隔离目录并做证据化验证：
1. 基座用与替身编译时完全同款的运行时（版本错一位都会拒收字节码）；
2. 部署前快照原版用户数据目录关键文件的修改时间；
3. 若主程序写死了用户数据路径，在入口 index.js 里做 setPath 拦截器，强制
   重定向到独立目录；
4. 启动后验证：进程树完整性 / 应用日志无异常 / 原版数据零触碰（对比时间戳）
   / 零外连（netstat）/ 替身文件确认真身（字节数）；
5. 验证"重启稳定性"：杀进程后立即重启，至少三轮；
6. 明确列出【已机器验证】与【需人工目验】两栏，不夸大。
```

---

## 15.7 远程通道搭建提示词（跨系统作战时用）

- **用途**：AI 在 Mac 上，目标软件在 Windows 实体机上——先打通"端口映射 + OpenSSH"自动化通道。
- **使用时机**：需要远程操控 Windows 执行编译/部署/验证，且 Windows 的 sshd 未就绪或损坏。

```text
我在用【远控软件】远程控制我的 Windows 电脑。我希望 AI 能通过它的端口映射 + OpenSSH 直接命令行
操控那台 Windows，不用我手动复制粘贴。你先打通这个通道：修复 Windows 上的 sshd，在 Mac 上验证
ssh 能通，并把"传文件、跑 PowerShell 脚本"两条通路都试通。
```

---

# 附录：全套代码模板（逐字可用，替换契约值与路径即用）

## 附录① 探针 main.js（配套 15.1）

```js
// 探针入口：跑在原厂 Electron 运行时里，动态加载原厂授权模块并逐个体检
const { app } = require('electron');
const fs = require('fs');
const path = require('path');
const DIR = 'C:\\Users\\Administrator\\AppData\\Local\\Temp\\decard-probe'; // ★ 改成你的绝对路径
const OUT = path.join(DIR, 'probe-exports.ndjson');
const rows = [];
const rec = (obj) => rows.push(JSON.stringify(obj));

app.whenReady().then(async () => {
  app.setPath('userData', path.join(DIR, 'userdata')); // 隔离，不碰真数据
  require(path.join(DIR, 'rt\\resources\\app\\node_modules\\bytenode')); // ★ 绝对路径加载
  const act = require(path.join(DIR, 'sandbox-copy\\core\\activation.jsc'));
  rec({ step: 'exports', keys: Object.keys(act) });        // ① 导出名单
  for (const k of Object.keys(act)) {                       // ② 逐个调用体检
    if (typeof act[k] !== 'function') { rec({ k, type: 'prop', v: act[k] }); continue; }
    try {
      const r = act[k]('TESTDUMMY', false);                 // 带假参数试形态
      if (r && typeof r.then === 'function') {              // 异步函数等结果
        const v = await r; rec({ k, async: true, ret: v });
      } else rec({ k, async: false, ret: r });
    } catch (e) { rec({ k, threw: String(e).slice(0, 200) }); }
  }
  fs.writeFileSync(OUT, rows.join('\n'));
  app.exit(0);                                              // ★ 主动退出，防挂死
});
```

探针包结构：`runtime\`（原厂运行时副本）+ `probe-app\`（探针入口）+ `sandbox-copy\`（授权模块字节码 + bytenode），整个目录拷到 Windows 双击即用，产出 `probe-exports.ndjson`。

**为什么这里必须绝对路径 require**：探针 main.js 与 bytenode 分处两个目录（`probe-app\` vs `runtime\resources\app\node_modules\`），相对解析必失败挂死。只有与 node_modules 同处 `resources\app\` 的部署版入口 index.js 才允许写 `require('bytenode')`（相对）——见 pitfalls.md 坑 11' 的适用边界。

## 附录② 挂钩观察 index.js（配套 15.2）

```js
// 观察副本 index.js：Module._load 挂钩，记录授权模块的每次真实调用
'use strict';
const fs = require('fs');
const path = require('path');
const Module = require('module');
const LOG = 'C:\\Users\\Administrator\\Downloads\\probe-out\\hook-obs.ndjson';  // ★ 改成你的路径
const rec = (o) => { try { fs.appendFileSync(LOG, JSON.stringify(Object.assign({ t: new Date().toISOString() }, o)) + '\n'); } catch (_) {} };

const origLoad = Module._load;
Module._load = function (request, parent, isMain) {
  const loaded = origLoad.apply(this, arguments);
  if (request.endsWith('activation.jsc')) {           // 目标模块出现
    rec({ step: 'load', request, parent: parent && parent.filename });
    for (const k of Object.keys(loaded)) {
      if (typeof loaded[k] !== 'function') { rec({ step: 'prop', k, v: loaded[k] }); continue; }
      const orig = loaded[k];
      loaded[k] = function () {                        // 包一层代理
        const args = Array.from(arguments);
        rec({ step: 'call', fn: k, args });
        const ret = orig.apply(this, arguments);
        if (ret && typeof ret.then === 'function') {   // 异步：等落定再记
          const t0 = Date.now();
          ret.then(v => rec({ step: 'ret-async', fn: k, ms: Date.now() - t0, returns: v }))
              .catch(e => rec({ step: 'ret-async-err', fn: k, err: String(e) }));
        } else rec({ step: 'ret', fn: k, ms: 0, returns: ret });
        return ret;
      };
    }
  }
  return loaded;
};
require('bytenode');
require('./main.jsc');                                  // 原厂主程序照旧运行
```

**挂钩日志要看什么**：load 记录（谁在加载授权模块）；主程序读了哪些常量属性；`setOnlineUrl` 是否被调（实锤"云地址由主程序注入"）；每个函数的**真实参数**（与探针假设比对）；异步函数的落定返回值与耗时；哪些导出从未被调用。

## 附录③ 替身模块 activation.js 完整源码（配套 15.4，替换契约值即用）

```js
'use strict';
/* 自用版授权模块 —— 替代原厂 activation.jsc（编译为同名 .jsc）
 * 契约依据：动态探针（导出清单）+ 挂钩观察（调用形态）+ 网页源码（消费结构）
 * 行为：本地自用授权，恒为已激活永久卡；零联网、零试用文件、零注册表 */
const crypto = require('crypto');
const fs = require('fs');
const os = require('os');
const path = require('path');
const { execSync } = require('child_process');

const TRIAL_DAYS = 3;                                    // 网页启动时会读，照抄实测值
const LICENSE_LABELS = {                                 // 档位标签表，照抄实测值
  trial: '试用版', monthly: '月卡', quarterly: '季度版',
  semiannual: '半年卡', yearly: '年卡', biennial: '2年卡', permanent: '永久卡'
};

/* —— 日志：仅异常/被忽略调用时写，不含凭据与激活码 —— */
function logLine(msg) {
  try {
    const dir = path.join(process.env.USERPROFILE || '.', 'AppData', 'Roaming',
                          'decard-local', 'logs');       // ★ 与隔离的用户数据目录一致
    fs.mkdirSync(dir, { recursive: true });
    const f = path.join(dir, 'activation-local.log');
    if (fs.existsSync(f) && fs.statSync(f).size > 100 * 1024) fs.renameSync(f, f + '.1');
    fs.appendFileSync(f, new Date().toISOString() + ' ' + msg + '\n');
  } catch (_) {}
}

/* —— 机器码：优先复刻原厂算法（注册表 MachineGuid 的 md5）—— */
function readMachineGuid() {
  try {
    const out = execSync('reg query "HKLM\\SOFTWARE\\Microsoft\\Cryptography" /v MachineGuid',
      { encoding: 'utf8', timeout: 5000, windowsHide: true });
    const m = out.match(/MachineGuid\s+REG_SZ\s+([0-9a-fA-F-]+)/);
    return m ? m[1].trim().toLowerCase() : null;
  } catch (_) { return null; }
}
function getMachineId() {
  let src = readMachineGuid();
  if (!src) {                                            // 读不到注册表时的兜底
    src = [os.hostname(), process.env.USERNAME || 'unknown'].join('|');
    logLine('WARN: MachineGuid unavailable, fallback used');
  }
  return crypto.createHash('md5').update(src, 'utf8').digest('hex');
}
function inviteCodeFor(machineId) {                      // 邀请码 = 机器码前 6 位大写
  return String(machineId || getMachineId()).slice(0, 6).toUpperCase();
}

/* —— 授权状态：恒为已激活。字段集与界面契约完全一致 —— */
function getLicenseState() {
  const machineId = getMachineId();
  return {
    activated: true, trial: false,
    licenseType: 'permanent', licenseLabel: '自用版', expire: '',   // ★ licenseType 用源码里被字面量比较的档位值
    machineId: machineId, inviteCode: inviteCodeFor(machineId), reason: ''
  };
}

function gateActivation() { return Promise.resolve({ ok: true }); }  // 门禁放行（异步形态）
let _onlineUrl = '';
function setOnlineUrl(url) { _onlineUrl = String(url || ''); }        // 收地址但永不联网
function loadActivation() { return null; }               // 无卡密文件，与原"无文件"同形态
function saveActivation() { return true; }               // 无需持久化，no-op 成功
function isActivated() { return true; }                  // 主进程内部判断点的保险丝

function verifyActivation(code) {                        // 诚实拒绝，不伪造成功
  logLine('verifyActivation ignored, codeLen=' + String(code || '').length);
  return { valid: false, reason: '自用版无需激活码' };
}
function submitActivation(code) {
  logLine('submitActivation ignored, codeLen=' + String(code || '').length);
  return { ok: false, error: '自用版无需激活码' };
}
function claimCard(code) {
  return { ok: false, error: '邀请码奖励不适用于自用版' };
}

module.exports = {
  getMachineId, inviteCodeFor, verifyActivation, loadActivation, saveActivation,
  isActivated, getLicenseState, submitActivation, claimCard,
  gateActivation, setOnlineUrl, TRIAL_DAYS, LICENSE_LABELS   // 导出数量与实测清单一致，一个不少
};
```

## 附录④ 编译 + 回环验证一体脚本 compile-main.js（配套 15.5）

```js
// compile-main.js：跑在目标 runtime 的 resources\app 里（应用目录模式）
const bytenode = require('bytenode');
const fs = require('fs');
const path = require('path');
const { app } = require('electron');
const DIR = 'C:\\…\\decard-local-build';                  // ★ 改成你的绝对路径
app.whenReady().then(async () => {
  const results = { steps: [] };
  try {
    bytenode.compileFile({ filename: path.join(DIR, 'activation.js'),
                           output: path.join(DIR, 'activation.jsc') });
    const m = require(path.join(DIR, 'activation.jsc'));   // ← 回环：编译产物立刻自验
    results.exportKeys = Object.keys(m);                   // ① 导出齐全
    results.licenseState = m.getLicenseState();            // ② 状态结构正确
    const g = m.gateActivation('D:\\<路径>');              // ③ 门禁异步 + 放行
    results.gateIsPromise = g && typeof g.then === 'function';
    results.gateResolved = await g;
    results.verify  = m.verifyActivation('TEST');
    results.submit  = m.submitActivation('TEST');
    results.machineId = m.getMachineId();                  // ④ 尝试同值对齐
  } catch (e) { results.fatal = String(e && e.stack || e); }
  fs.writeFileSync(path.join(DIR, 'compile-results.ndjson'), JSON.stringify(results));
  app.exit(0);                                             // ★ 主动退出
});
```

**回环验证通过标准**（案例实测）：产物字节数与预期一致（案例 7,080 字节）✅；导出对账缺失清单为空 ✅；门禁是 Promise 且落定 `{ok:true}` ✅；授权状态结构逐项一致 ✅；各拒绝函数文案 ✅；机器码同值 ➖（不一致则按"接受新指纹"决策记录）。

## 附录⑤ 入口 index.js setPath 拦截器（配套 15.6）

```js
// 入口 index.js（明文，自用版的第一行代码）：数据目录双重隔离
'use strict';
const path = require('path');
try {
  const { app } = require('electron');
  const LOCAL = path.join(app.getPath('appData'), 'decard-local');   // ① 预置：独立目录名，避开原厂
  app.setPath('userData', LOCAL);
  const orig = app.setPath.bind(app);
  app.setPath = function (name, p) {                                // ② 拦截器
    if (String(name).toLowerCase() === 'userdata') p = LOCAL;       //    主程序写死也重定向
    return orig(name, p);
  };
} catch (_) {}
require('bytenode');
require('./main.jsc');
```
