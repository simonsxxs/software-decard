# 发布记录：software-decard

> 每发布一个平台就更新一行。状态：✅已上架 / ⏳审核中 / ❌失败（写原因）/ ➖不适用

| 平台 | 状态 | 链接 | 发布日期 | 备注 |
|------|------|------|---------|------|
| ima 技能广场 | ➖ 不适用 | | | 用户决定不上传 |
| GitHub | ✅ 已上架 | https://github.com/simonsxxs/software-decard | 2026-10-12 | 专属仓，主开发副本 |
| Gitee | ✅ 已上架 | https://gitee.com/simonsxx/software-decard | 2026-10-12 | GitHub 同步镜像 |
| SkillHub | ⏳ 审核中 | https://skillhub.cn/skill/software-decard | 2026-10-12 | GitHub 导入发布（第二次：自定义图标 + 带署名描述），API 201；安全审核中，通过后公开页生效 |
| aiSkill.market | | | | |
| ClawHub | | | | |
| 扣子（豆包） | | | | |
| 腾讯元器 | | | | |
| 文心智能体 | | | | |
| ChatGPT GPTs | | | | 需 Plus |

## 统一物料

- 发布名：software-decard 软件去卡密自用版
- 简介（短，50字）：把自己购买的 Electron 软件改造成免卡密自用版：不改本体，用"诚实的替身授权模块"换掉授权校验。
- 简介（长，200字）：一套可复用的软件去卡密方法论 + 操作手册 + 分步 AI 提示词。核心路线：侦察软件形态 → 写探针摸授权链路 → 挂钩/替身替换授权校验 → 编译验证 → 界面注入署名层。不改软件本体一个字节，不碰作者私钥，不伪造激活凭证；仅限自购软件自用，绝不分发。含 40+ 条踩坑表、远程操控 Windows 手册、9 段可直接粘贴的分步提示词、白纸实测报告。
- 分类标签：开发编程（SkillHub 导入流程未要求必选分类，当前"未分类"，如需补充分类可在审核通过后更新）
- 使用示例（3条）：
  1. 帮我把这个自购软件去掉卡密，改成自用版
  2. 侦察一下这个 Electron 软件的授权校验链路
  3. 按 SKILL.md 第 7 步部署 ui-patch 界面署名层
- 头像/封面：自定义图标 assets/skillhub-icon.png（锁形图标，已上传，合规审核后公开展示）

## SkillHub 发布备注（本次实战沉淀）

- 路径：发布 Skill（最快上架）→ 从 GitHub 导入 → 勾选仓库根目录的 SKILL.md → 下一步确认信息 → 发布
- GitHub 账号需预先绑定（仪表盘左侧显示"已绑定"）；导入列表自动扫描全部公开仓库的 SKILL.md
- 本地上传路径的限制：压缩包**只允许 .md 文件**（LICENSE/js/css/png 会 400「不允许的文件类型」）；WebBridge 的 `upload` 动作还需要浏览器扩展开启"允许访问文件网址"，本次未开启，故走 GitHub 导入
- 提交是否成功看 `api.skillhub.cn/api/v1/community/skills/publish` 响应：**201 = 成功**，400 看响应体
- 导入流程的表单**没有分类必选**（与手动上传不同），发布后在"我的 Skills"显示"安全审核中"
- 坑：发布成功后页面跳转 dashboard，若此时复用旧 ref 点击，可能误触列表里其他 Skill 的"更新"弹窗——看到非目标 Skill 的对话框直接关闭，不要提交
- 审核中的 Skill 没有「更新」入口（只有已发布的才有）；要改图标/描述只能取消发布重新提交
- 描述字段标了必填（红框），尽管 placeholder 说留空自动用 SKILL.md frontmatter——**一定要手动填**，并带上署名
- 文件注入替代方案（扩展没开文件访问权限时）：把文件 push 到 GitHub 仓库 → 页面里 `fetch('https://raw.githubusercontent.com/<owner>/<repo>/main/<path>')`（CORS 放行）→ 构造 File → DataTransfer 赋给 `input.files` → dispatch change。图标上传走「自定义」tab → 点击上传图片（弹出裁剪对话框）→ 点「上传」确认
- 发布后内容以 GitHub 仓库当前状态为准；后续更新 Skill 走"更新"重新触发审核
