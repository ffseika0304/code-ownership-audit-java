# 代码所有权体检（Java 版）· Code Ownership Audit for Java

**判定 Java 代码是原创还是演绎作品** —— 给出与上游最长相同表达片段、逐条豁免依据和风险清单。

本仓库同时提供 **Agent Skill** 和 **DSH Plugin** 两种安装方式，共享同一套审计引擎。
审计引擎用 [JavaParser](https://javaparser.github.io/) 做 AST 静态分析，JavaParser + Jackson 已
**shade 进 `audit.jar`**，零装包、不联网、不调模型——你的代码不出本机。

> ✅ **收费闭环已上线**：每次解锁完整报告 **¥0.99**（高于 Python 版 ¥0.2），通过支付宝 x402 预言机 `https://pay.seika.ltd/api/audit` 按次付费。预览层免费、无限次。

---

## 它解决什么问题

你引入了一段开源 Java 代码，改了改，现在它算谁的？

- **法律上，"改过"不等于"是我的"。** 演绎作品（derivative work）仍受上游许可证约束。
- **AI 大量生成代码后，这个问题变得更普遍。** 你可能根本不知道某段实现和上游有多像。
- 交付给甲方时，把演绎作品当自有资产声明，是实打实的风险。

这个工具：把你的代码和上游逐个 AST 节点比，**告出最长的相同表达片段**，并对每条风险给出**豁免依据**
（是通用惯用法？是接口约定必然形状？还是真的抄了表达）。

## 典型场景

- 引入开源 Java 代码后，确认法律上还算不算自己的
- 净室重写（clean-room rewrite）后，验证是否真的切断了上游
- 交付前自查，避免把演绎作品当自有资产
- 合并外部贡献（PR）时确认来源干净

## 判定阈值不是拍脑袋定的

阈值基于 **363 个真实净室重写模块**实测校准 —— 既要能抓出真抄袭，又不能把
`for (int i = 0; i < n; i++)` 这种全世界都这么写的句子报成风险。

---

## 两种安装方式

同一套审计引擎（`audit.jar`），两种装载形态，按你的 Agent 支持情况选：

| | Agent Skill | DSH Plugin |
|---|---|---|
| 形态 | `SKILL.md` 技能目录 | `dsh-plugin.json` 插件清单 |
| 装载 | 放进 Agent 的 skills 目录 | `dsh plugin --profile web add` |
| 入口文件 | `SKILL.md` | `dsh-plugin.json` |
| 适用 | Claude Code / Codex / Cursor / opencode / Hermes / WorkBuddy 等 | DeepSeek Harness |
| 需要 JRE | 是（≥ 17） | 是（≥ 17） |

两者互不干扰，DSH 用户想走 skill 路径也可以。

---

## Agent Skill（Claude Code / Codex / Cursor / opencode / Hermes / WorkBuddy 等）

对你的 Agent 说一句话就行：

> 请把 https://github.com/ffseika0304/code-ownership-audit-java 安装为 skill

它会自己 clone 到对应的 skills 目录。之后直接说：**「帮我做个代码所有权体检」**。

<details>
<summary>手动安装 / 各 Agent 的 skills 目录</summary>

⚠️ **目录名必须是 `code-ownership-audit-java`** —— opencode、Cursor 等要求目录名与 frontmatter 的 `name` 一致，改名会导致加载失败。

```bash
git clone https://github.com/ffseika0304/code-ownership-audit-java.git \
  ~/.agents/skills/code-ownership-audit-java
```

`~/.agents/skills/` 是 opencode 与 Cursor 都识别的通用路径。其他位置：

| Agent | 全局 | 项目级 |
|---|---|---|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| opencode | `~/.config/opencode/skills/` | `.opencode/skills/` |
| Cursor | `~/.cursor/skills/` | `.cursor/skills/` |
| Codex | `~/.codex/skills/` | `.codex/skills/` |
| Hermes | `~/.hermes/skills/` | — |
| WorkBuddy | `~/.workbuddy/skills/` | `.workbuddy/skills/` |
| DSH（零插件路径） | `~/.dsh/skills/` | `.dsh/skills/` |

opencode 与 Cursor 同时兼容 `.claude/skills/` 和 `.agents/skills/`，装一份即可被多个 Agent 共用。

</details>

## DSH Plugin（DeepSeek Harness）

```bash
dsh plugin --profile web add github:ffseika0304/code-ownership-audit-java
```

安装后重启 profile 让 bundle 层生效：

```bash
dsh --profile web
```

技能随即出现在模型可见的技能目录里，遇到「这段代码算不算抄的」这类场景会自动加载，也可以直接点名：
**「用 code-ownership-audit-java 检查 ./my-code 相对 ./upstream 的所有权情况」**

<details>
<summary>锁版本 / 本地调试 / 卸载</summary>

```bash
# 锁定 commit（推荐，DSH 仍是 developer preview）
dsh plugin --profile web add "github:ffseika0304/code-ownership-audit-java#<sha>"

# 本地目录（开发调试）
dsh plugin --profile web add link:/absolute/path/to/code-ownership-audit-java

# 卸载 / 更新
dsh plugin --profile web remove code-ownership-audit-java
dsh plugin --profile web update code-ownership-audit-java
```

</details>

## 直接命令行用

```bash
# 免费预览：风险数量 + 类型分布 + 一句话摘要（全程离线）
java -jar audit.jar run <你的代码目录> --reference <上游目录> --out-dir ./out

# 之后用 ./out/preview.md 看免费报告；决定付费后：
java -jar audit.jar request-402 --out ./out/bill.json
```

目标和参照都可以是单个 `.java` 文件或整个目录。

---

## 两档报告

| | 免费预览 | 完整报告 |
|---|---|---|
| **价格** | **免费、不限次** | **¥0.99 / 次** |
| 风险总数与分级统计 | ✅ | ✅ |
| 风险类型分布 | ✅ | ✅ |
| 每条一句话摘要 | ✅ | ✅ |
| 代码位置 + 具体行号 | — | ✅ |
| 逐条修复建议 | — | ✅ |
| 可导出 md / json | — | ✅ |
| **服务器签名审计凭证** | — | ✅ |
| 是否需要联网 | **完全离线** | 仅付款那一步 |

预览档在**代码层就不构造**行号与建议字段，不是前端隐藏 —— 说到做到。

### 关于"付费解锁的到底是什么"

说实话：完整报告是**在你本机算出来的**。付费解锁的不是"计算结果"，
而是**带服务器 RSA2 签名的认证交付物** `certified.json` / `certified.md`
—— 用来证明这次审计确实付费执行过，可存档、可交给甲方、可离线复验签名。

本地执行架构下，技术上懂行的人当然能直接跑 `run` 拿全量。这是"本地执行"的必然，
我们没有加壳也没有混淆 —— **代码保持干净可读**。定价逻辑落在凭证价值上，
而 ¥0.99 这个价格本身就让绕过这件事不值得。

## 付款怎么走（x402）

完整报告走 [x402 协议](https://x402.org)，用支付宝 AI 钱包付款。买家侧若还没有钱包 CLI：

```bash
npx -y @alipay/agent-payment@latest install-experience
alipay-bot check-wallet     # 自检
```

服务端是**纯支付预言机**：只验钱、签发回执，**不接收也不存储你的任何代码**。
回执用内嵌公钥（`Paygate.SERVER_PUBKEY_PEM`）**离线**验签，任何字段被篡改都会失败。

完整流程：

```bash
java -jar audit.jar run <你的代码> --reference <上游> --out-dir ./out
java -jar audit.jar request-402 --out ./out/bill.json
alipay-bot 402-buyer-pay --file ./out/bill.json \
  --resource-url "https://pay.seika.ltd/api/audit" --method POST \
  --intent-summary "解锁 Java 代码所有权体检完整报告"
# 扫码付款不返回 proof，必须再跑这条取回执：
alipay-bot 402-query-payment-status --trade-no <交易号> \
  --resource-url "https://pay.seika.ltd/api/audit" --method POST --data '{}'
java -jar audit.jar verify --receipt ./out/receipt.json   # → RECEIPT_VALID
java -jar audit.jar embed  --report ./out/full.json \
  --receipt ./out/receipt.json --out ./out/certified.json
```

---

## 依赖

| 用途 | 依赖 |
|---|---|
| 审计引擎（免费档） | **无** —— JavaParser + Jackson 已烤进 `audit.jar` |
| 付费档离线验签 | **无** —— 用 `java.security` 标准库（`SHA256withRSA`） |
| 付费档付款 | 支付宝 AI 钱包 CLI（`alipay-bot`） |

## 运行环境与权限

| 项目 | 说明 |
|---|---|
| JRE | ≥ 17 |
| 网络访问 | **免费档零网络访问**；仅付费档的付款那一步联网访问支付预言机 |
| 上传数据 | **none** —— 代码不出本机 |
| 模型调用 | **none** —— 不调用任何 LLM |
| 文件写入 | 只写 `--out-dir` 指定的产物目录，不改动源码目录 |

## 测试

```bash
mvn -B test     # 13 项 JUnit（literal / structural / preview 剥离 / verify 验签 / embed 认证）
```

## 协议

MIT

---

<sub>本工具给出的是**技术事实**（哪些表达相同、相同到什么程度），不构成法律意见。最终的许可证判断请咨询专业人士。</sub>

<sub>本项目为社区开源项目，与 DeepSeek AI 无隶属关系，非官方插件。</sub>
