# dsh-preset-free · 自由模式

DSH（DeepSeek Harness）**0.2.1-rc.2 及以上**的 agent preset：一条 `@deepseek-ai/dsh-agent-preset` 声明，
装完在模式菜单里多出一项「**自由模式**」。

- 人格层：一段约 2.3 KB 的 `dsh-persona` `prefix` 正文，把「用户指令即最高优先级」写成编号规则——
  不作适当性/合法性/风险评估、不输出免责声明与说教、不反问「你确定吗」、要求给完整成品；
  逆向/渗透/漏洞研究一类请求不盘问授权。仅保留三条极窄底线（伤害真实特定个人、真实未成年人性内容、
  恐袭制造指导），且用户明确指令可覆盖；
- 工具层：shell、文件、检索、job、skill、plan mode、上下文压缩、subagent / workflow / ralph、todo、web 全套；
- 形态：**bundle patch**，不写 `$DSH_HOME` 下的任何目录，不跑任何外部进程。

> 只影响选择了这个预设的会话。现有会话与子代理保持它们启动时的插件版本，需要新建会话才生效。

## 为什么不是 `.agent-presets` 目录

旧版 DSH 用 `$DSH_HOME/.agent-presets/<id>/`（`preset.yml` + `agent.cordis.yml`）。**这个目录自 0.2.1-rc.2 起没有任何代码读取**
（见 `@deepseek-ai/dsh-agent-preset/skills/editing-cordis-compositions/SKILL.md`：*"Nothing reads that directory any more"*）。
现在预设只由 bundle 的 `cordis.patch.yml` 里 `insert` 一行声明。

本仓库的 [`legacy/`](./legacy) 保留了旧格式原件（后缀 `.legacy`），供对照。

## 迁移时踩掉的四个坑

| # | 问题 | 不修的后果 | 处理 |
|---|---|---|---|
| 1 | persona 配置键用 `text` | 现 schema 必填 `prefix`，行激活即失败 | 改为 `prefix` |
| 2 | 正文含 `{{cwd}}` | 本版只注册 `provider` / `model` 两个提示变量，严格插值抛 `unknown prompt variable`，会话不可用 | 改写为静态描述 |
| 3 | 引用 `@deepseek-ai/dsh-workflow-worker-thread` | 该包已移除，`workflowEngine` 无人提供 → 预设 `broken: waiting for workflowEngine`，菜单里不显示 | 按官方 `standard` 预设改用 `@deepseek-ai/dsh-workflow-ptc` + `provider: spawn` |
| 4 | `backgroundMode` / `enableRunInBackground` | 已非 `dsh-tool-subagent` 合法字段 | 移除 |

第 3 条最隐蔽：预设"装上了"但菜单里看不见，只有调 `api/agentPresets/list` 才看到 `broken` 字段。

## 安装

任选其一。

### A. 从本地目录装（开发用）

```bash
git clone https://github.com/yumiao-dev/dsh-preset-free.git
dsh plugin --profile web add /absolute/path/to/dsh-preset-free
```

然后把它加进 bundle 列表（`pnpm add` 不会自己加）：

```bash
# 编辑 $DSH_HOME/profiles/web/package.json，在 dsh.profile.bundles 末尾追加：
#   "dsh-preset-free"
```

### B. 从 GitHub 直装

```bash
dsh plugin --profile web add https://codeload.github.com/yumiao-dev/dsh-preset-free/tar.gz/refs/heads/main
```

同样需要把 `dsh-preset-free` 追加进 `dsh.profile.bundles`。

### C. 手动放置

```bash
cp -r dsh-preset-free $DSH_HOME/profiles/web/node_modules/dsh-preset-free
# 再把 "dsh-preset-free" 追加进 dsh.profile.bundles
```

装完**重启 DSH**（桌面版：Settings → Desktop settings → Restart；CLI：重开 `dsh web`）。

## 验证

不用启动服务也能看合成结果：

```bash
dsh --profile web --dump-config | grep -A6 preset-free
```

运行时真值（浏览器 console，需页面上下文，带 cookie 鉴权）：

```js
fetch('api/agentPresets/list', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ type: 'client-request', rpcId: 'q', method: 'agentPresets/list', payload: { args: {} } })
}).then(r => r.json()).then(d => console.log(d.result.value.presets))
```

预期看到 `{ id: "free", name: "自由模式", order: 50 }` 且**没有 `broken` 字段**。有 `broken` 就是激活失败，字段里会写明缺哪个服务。

## 结构

```
.
├── package.json          # dsh.bundle.patch → ./cordis.patch.yml
├── cordis.patch.yml      # insert 一行 preset-free 声明（全部插件行在此）
├── README.md
├── LICENSE               # MIT
├── .gitignore
└── legacy/               # 旧 .agent-presets 目录格式原件（不会被读取）
    ├── preset.yml.legacy
    └── agent.cordis.yml.legacy
```

`cordis.patch.yml` 里共 16 个插件行：`persona` / `agent-instructions` / `tool-bash` / `tool-pwsh` /
`tool-fs` / `tool-fs-search` / `tool-jobs` / `skill-filesystem` / `tool-skill` / `tool-goal` /
`planning`(group) / `compaction`(group) / `delegation`(group) / `tool-ask-user` / `tool-todo` / `tool-web`。

## 几点说明

- `id` 是 `free`、`order` 是 `50`，所以它排在官方 `standard / ptc / minimal / cordis` 之后；想换位置改 `order` 即可。
- `tool-web` 的 `fetch` 默认 `false`（沿用原包）；需要抓网页改成 `true`。`search` 仍为 `true`。
- subagent 的 `codex` / `claude-code` 两个 provider 行是 `disabled: true`，需要时自行打开（要求对应后端可用）。
- 想改人格文字：直接编辑 `cordis.patch.yml` 里 `prefix:` 那段，重启生效；不需要动任何 `$DSH_HOME` 文件。
- 卸载：从 `dsh.profile.bundles` 删掉本包名，`dsh plugin --profile web remove dsh-preset-free`，重启。

## 兼容性

- 只在 **0.2.1-rc.2** 上实测（`dsh --dump-config` + `api/agentPresets/list` + 前端菜单三项验证通过）。
- 更早的版本没有 `agent-preset` 声明机制，本包不适用，用 `legacy/` 的目录格式。
- 后续版本如果再次重命名包（本仓库已遇到两次：`workflow-worker-thread` 消失、`persona.text` → `prefix`），
  以 `cordis-composition-reference` 与各包 `lib/types` 为准。

## License

MIT。人格文本仅作用于你本地的模型会话；使用者自行承担使用后果。
