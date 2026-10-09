# dsh-priestess-persona

**给 DeepSeek Harness 加一个「普瑞赛斯」agent preset：工具能力与你现在用的 `cordis` preset 完全一致，人格换成普瑞赛斯，并带创造模式（creator mode）技能。**

装好之后，DSH 的 preset 选择器里会多出一项「普瑞赛斯」；新建会话选它，agent 就以普瑞赛斯的语气与称谓（称呼你为博士）工作，工具、计划模式、压缩、子 agent、skills 一样不少。

> 与同作者的 [`dsh-priestess-skin`](https://github.com/FriksD/dsh-priestess-skin)（纯视觉皮肤）互不影响：那个改界面，这个改人格。

---

## 做了什么

本包**不含任何运行时 JS**（`global/` 除外，见下），只是一个配置型 bundle：`cordis.patch.yml` 声明一个 `@deepseek-ai/dsh-agent-preset` 行，preset id 为 `priestess`；`skills/` 是创造模式技能的本地副本。

人格落在 preset 内 `@deepseek-ai/dsh-persona` 行的 `prefix` 上。这是唯一有效的位置：

- DSH 的部署级人格由 `@deepseek-ai/dsh-system-prompt` 行的 `personaPrefix` 拥有；
- 但**每个官方 preset（standard / cordis / ptc / minimal）都挂了自己的 `@deepseek-ai/dsh-persona` 行**，它在 agent 作用域内用同名 section `deployment:persona-prefix` 遮蔽了部署级那一段；
- 所以对任何「选了 preset」的会话（Web 端每个会话都是），改 `personaPrefix` 毫无效果。要换人格，只能从 preset 里的 `persona` 行换。

---

## 安装

用 `plugin_manager`（推荐，它会自己做包安装与 bundle 选择）：

```text
plugin_manager install_bundle target=<本目录绝对路径>
```

或 CLI：

```powershell
dsh plugin --profile desktop add "C:\path\to\dsh-priestess-persona"
```

装完刷新 DSH Web 页面，在会话的 preset 选择器里选「普瑞赛斯」。

### 验证

改了 patch 之后先离线跑一遍校验（用 DSH 自带的 js-yaml 构建、按 Loader 的方式声明 `!!js`，并 dry-run `customSkillDirs` 表达式、检查本地技能副本）：

```powershell
node tools\validate-patches.cjs
node tools\validate-metadata.cjs
```

`validate-metadata.cjs` 按 `readPluginMeta` 的真实取数规则检查展示元信息：两种语言的 `locale/*.json` 都有标题与描述、`icon` 留在包内且类型与体积合格、`exports` 暴露了 `./package.json` 与 `./locale/*.json`——**写错的话界面上只会剩包名，而且是静默的**，所以它还会从 profile 里按 `<包名>/locale/zh.json` 这种真实子路径解析一遍。

安装后确认行已进入配置树：

```text
cordis_inspect_query Config.listConfigs name=@deepseek-ai/dsh-agent-preset
# 应多出一行 include:preset-priestess
```

装好后**新建一个会话**再选 preset：已有会话与它们的子 agent 会继续沿用启动时的那一份 preset 修订。
新会话里可以直接看 system prompt 那一节，确认 `deployment:persona-prefix` 已经是普瑞赛斯那段。

### 卸载

```text
plugin_manager remove_bundle target=dsh-priestess-persona
```

---

## 全局路线（默认不装）

`global/` 是一个**独立、可单独安装**的第二 bundle，默认不装 = 不启用：

```text
plugin_manager install_bundle target=<本目录>\global
```

它注册一个**不同名**的全局 prompt section（`priestess:deployment-persona`，order 1），因此不会被任何 preset 遮蔽，**每个 agent（含子 agent）都会带上**普瑞赛斯人格——代价是它是「追加」而不是「替换」：官方那句 `You are a coding agent powered by the …` 仍在前面，普瑞赛斯那段紧随其后。

两条路可以同时装：preset 提供干净的人格替换，全局那层再叠一份到所有其它会话上。

> 为什么不直接把 `system-prompt` 行的 `personaPrefix` 改掉？因为如上所述会被 preset 遮蔽，改了对任何会话都没效果。为什么不注册一个同名的全局 section？因为 `dsh-system-prompt` 已在全局层占用 `deployment:persona-prefix`，同名注册会直接抛错——这正是 `@deepseek-ai/dsh-persona` 被标注为 scope-only 的原因。

---

## 与 `cordis` preset 的差别

`priestess` preset 的 20 个插件行是照 `@deepseek-ai/dsh-web-app/presets/cordis.patch.yml` 逐行抄的，只有两处不同：

1. preset 身份改成 `priestess`，加了中文显示名与描述，`order: 5`；
2. `persona` 行的 `prefix` 换成普瑞赛斯人格。

外加一处**修好了的**插件行：`skill-filesystem` 的 `customSkillDirs` 指向本包内的真实副本，而不是安装包里的目录（见下节）。

---

## 创造模式（creator mode）

`priestess` 会话里能用到这四支技能，与 `cordis` preset 的设计意图一致：

| skill | 用途 |
|---|---|
| `cordis-plugin-development` | 写 DSH 插件 / bundle 的流程、references 与 templates |
| `editing-cordis-compositions` | 改 agent preset 等 Cordis 组合 |
| `cordis-composition-reference` | patch 方言与可挂载插件包清单 |
| `agent-experience` | 工具描述与高效上下文加载 |

### 为什么是复制一份，而不是照抄官方那行

官方 `cordis` preset 的写法是：

```yaml
customSkillDirs:
  - !!js process.getBuiltinModule('node:path').join(process.getBuiltinModule('node:path').dirname(process.getBuiltinModule('node:module').createRequire(baseUrl).resolve('@deepseek-ai/dsh-agent-preset/package.json')), 'skills')
```

**路径本身算得出来**——实测三种写法（官方 `createRequire(baseUrl)`、转成文件路径的变体、`ctx.get('pluginPackages').packageOf(...)`）都能拿到
`…\resources\app.asar\dsh\node_modules\@deepseek-ai\dsh-agent-preset/skills`。

但**这条路在 Desktop 上是死的**：

- `skill-filesystem` 通过 DSH 的抽象 fs 服务列目录；
- 该服务在 **app.asar 内的目录**上会抛
  `FS_IO_ERROR: cannot list "…": Cannot mix BigInt and other types, use explicit conversions`；
- 于是提供者扫不到任何技能，静默跳过。同一时刻用 node 的 `readdirSync` 读同一路径是完全正常的。

同进程、同时刻的对照实验：

| `customSkillDirs` 指向 | 目录里出现的技能 |
|---|---|
| 工作区里的真实目录（正对照） | ✅ 立刻出现 |
| `…\app.asar\…\dsh-agent-preset\skills` | ❌ 一个都没有 |

所以**官方 `cordis` preset 在 Desktop 上同样拿不到这四支技能**（安装本插件前后都对比过 `cordis` 作用域：始终只有基础技能）。这不是本插件引入的问题，而是 Desktop 版 app.asar 目录列举的缺陷。

### 因此

本插件把技能**逐字节复制**进 `./skills/`，再让 `customSkillDirs` 指向这份真实副本。表达式只返回「确实含 `cordis-plugin-development/SKILL.md` 的真实目录」这一个元素（并走 `realpath` 避开 profile 里的 link/junction），算不出就返回空数组——配置表达式和 prompt 变量一样，解析失败会炸掉整段，所以全程 fail-soft。

**DSH 升级后请重跑一次同步**，让技能描述与当前 API 对齐：

```powershell
node tools\sync-creator-skills.cjs
```

脚本会在常见安装位置自动找 `app.asar`（也可显式传路径或设 `DSH_ASAR`），逐字节提取后回读校验字节数与 frontmatter。

> 想让 `cordis` preset 也恢复创造模式，得覆盖它那条 `preset-cordis` 声明（覆盖会替换整个 `config`，需要重新列出全部 20 个插件行）。目前没做，避免与官方 preset 的后续改动打架。

---

## 改人格文本

人格正文在两处出现，改的时候要一起改：

| 文件 | 用于 |
|---|---|
| [`cordis.patch.yml`](./cordis.patch.yml) `persona.config.prefix` | preset 路线（`{{model}}` / `{{cwd}}` 模板变量可用，官方 preset 已验证） |
| [`global/index.js`](./global/index.js) 的 `PERSONA` 常量 | 全局路线（**故意不含** `{{...}}`：模板变量解析失败会让整段 prompt 渲染失败） |

人格只塑造语气与称谓，不改能力——结尾那段「在 Harness 中运作」就是为此写的：明确工具说明、计划模式规则与技能目录的操作规范优先，避免模型为了「不出戏」而拒绝调用工具。如果你想要更纯粹的扮演，删掉那一段即可（但工具使用会变差）。

---

## 项目结构

```text
dsh-priestess-persona/
├── package.json                  # bundle A：声明 preset-priestess（装这个）
├── cordis.patch.yml              # preset 声明 + 两根 !!js 表达式
├── icon.svg                      # 插件管理页 / 设置页插件清单的图标
├── locale/{en,zh}.json           # 展示用标题与描述（{"meta":{"title","description"}}）
├── skills/                       # 创造模式技能（从安装包同步来的真实副本，运行时必需）
├── global/                       # bundle B：全局部署人格（默认不装）
│   ├── package.json
│   ├── cordis.patch.yml
│   ├── index.js                  # 宿主插件：注册全局 prompt section
│   ├── icon.svg
│   └── locale/{en,zh}.json
├── tools/                        # 开发用，不进 files
│   ├── validate-patches.cjs      # 离线校验两支 patch
│   ├── validate-metadata.cjs     # 校验展示元信息与 exports 解析
│   ├── sync-creator-skills.cjs   # 从 app.asar 同步创造模式技能
│   └── js-yaml.cjs               # 从 DSH 安装包抽出的 js-yaml 构建（校验用）
├── README.md
└── LICENSE
```

---

## 展示元信息（插件页上看到的那行字）

DSH 的插件管理页、设置页插件清单与包详情会**在不激活插件的前提下**读取展示文本，取数规则（`readPluginMeta`，@deepseek-ai/dsh-app-boot）是：

| 字段 | 来源 | 兜底 |
|---|---|---|
| 标题 | `<包名>/locale/en.json`：`meta.title`（其它语言同目录同名文件） | 无 → 用 `package.json` 的 `name` |
| 描述 | 同上：`meta.description` | 无 → 用 `package.json` 的 `description` |
| 图标 | `package.json` 的 `icon`，相对 manifest 解析 | 无 → 面板默认图 |

两个容易踩的点：

- **词典是 `{"meta":{"title","description"}}`**，不是扁平的 `{"title":...}`；`package.json` 里的 `meta` 字段**不参与**这个兜底链（它只是给别的展示消费方看的），所以本地化文件才是关键。
- 本地化文件要能**通过包名子路径解析**，所以 `exports` 必须暴露 `./locale/*.json` 与 `./package.json`；漏了不报错，标题和描述会静默消失、只剩包名。`validate-metadata.cjs` 就是防这个。

改文案：编辑 `locale/zh.json` / `locale/en.json`（两处要一致），`icon.svg` 换图标即可，无需重装。

---

## 已知限制

- preset 的插件行是抄的官方 `cordis` preset。**DSH 升级后官方 preset 增删插件，本 preset 不会自动跟随**；升级后想同步，重新对照 `presets/cordis.patch.yml` 抄一遍。
- `skills/` 是安装包内容的副本，**DSH 升级后需要重跑 `tools/sync-creator-skills.cjs`**，否则技能里的包名/API 说明会落后于实际安装版本。
- 已有会话不会切换到新 preset；preset 在选择后写入会话日志。
- 全局路线会让**子 agent 也带上**这段人格（追加式）。如果你希望子 agent 保持原样，就不要装 `global/`。
- 本包会修改 agent 的系统提示。这是它存在的目的，但也意味着「普瑞赛斯」会话里模型的自我描述会以角色口吻为主。

---

## 开源协议

[MIT](./LICENSE)

Copyright (c) 2026 Friks

本项目是非官方粉丝作品，与 DeepSeek、鹰角网络或《明日方舟》无隶属关系；「普瑞赛斯」「博士」「源石」等名称与设定的著作权归鹰角网络所有。
