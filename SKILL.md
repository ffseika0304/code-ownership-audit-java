---
name: code-ownership-audit-java
description: 判定 Java 代码是原创还是演绎作品，给出与上游最长相同表达片段、逐条豁免依据和风险清单。纯本地 AST 分析，依赖烤进 jar，不联网、不调模型。
slug: code-ownership-audit-java
displayName: 代码所有权体检（Java 版）
version: 1.0.0
summary: 判定 Java 代码是原创还是演绎作品，给出与上游最长相同表达片段、逐条豁免依据和风险清单
license: MIT
---

# 代码所有权体检（Java 版）

判断一份 **Java** 代码相对某个上游是**原创作品**还是**演绎作品**，并说明依据。
这是 Python 版 `code-ownership-audit` 的 Java 移植：审计引擎用 [JavaParser](https://javaparser.github.io/) 重写，
依赖全部 **shade 进 `audit.jar`**，用户侧零装包——只要有一份 **JRE 17+** 就能 `java -jar` 一条命令跑。

> ✅ **收费闭环已上线**：每次解锁完整报告 **¥0.99**（高于 Python 版 ¥0.2），通过支付宝 x402 预言机 `https://pay.seika.ltd/api/audit` 按次付费，客户端自动带 `resource_id=/api/audit/java` 路由到支付宝服务 `API_12E012B1842A4F20`。预览层免费、无限次。

## 什么时候用

- 引入了开源 Java 代码、改了一遍，想知道法律上还算不算别人的
- 做了净室重写，要验证真的切断了上游
- 交付前自查，避免把演绎作品当自有资产卖
- 合并外部贡献，想确认来源干净

## 它怎么判

按「实质性相似」判定，而不是逐字节比对。核心是区分**表达**与**非表达**：

同一件事只有一种写法时，写得一样不构成抄袭。以下 8 类识别为非表达，不计入相似度：

| 类别 | Java 例子 | 为什么豁免 |
|---|---|---|
| `import` / `package` | `import java.util.*` | 名字由依赖/包结构决定，改了就找不到 |
| `signature` | `public int run(String x)` | 公开接口。API 名称不受版权保护，改了破坏调用方 |
| `annotation` | `@Override` / `@Service` | 绑定框架定义的名字 |
| `control_flow` | `for (...)` / `if (...)` | 控制流只有少数写法 |
| `keyword` | `return` / `throw` / `break` | 语言语法，无替代拼写 |
| `type_decl` | `class` / `interface` / `enum` | 公开类型契约 |
| `field_binding` | `this.name = name` | 构造器存字段的惯用写法 |
| `assert` / 字面量常量 | `assert x == 1` | 陈述公开契约 |

剩下的算表达代码。**连续 3 行以上完全相同**（或结构层连续 3 条以上语句形状相同）即判定为演绎作品。

另外，字节完全相同的文件直接判演绎 —— 因为表达行可能全被签名行分隔，单看片段长度会漏判。

## 收费模式（免费预览 / 付费完整）

降低用户决策成本，也直接展示价值，而不是靠「承诺」吸引付费：

- **免费预览报告**（不联网、不付款）：风险数量统计、风险类型分布、每条风险的一句话摘要。
  **不显示代码位置、不显示行号、不显示修复建议。**
- **付费完整报告**（付款后解锁）：每条风险的详细分析（代码位置、具体行号、风险描述）、
  具体修复建议、可导出的 Markdown/JSON、可存档的审计凭证（服务器签名回执）。

### 定价（已定稿）

| 档位 | 价格 | 计费方式 |
|---|---|---|
| 预览报告 | **免费** | 本机离线跑，无限次 |
| 完整报告 | **¥0.99 / 次** | 按调用计费，一次一单 |

> 架构文档 §0 要求「Java 定价高于 Python 版（¥0.2）」，最终定为 **¥0.99**（已与用户确认）。
> 真实金额以服务端 `AIPAY_AMOUNT=0.99` 签发的 402 账单 + 回执校验为准，不受客户端常量影响。

风险等级按连续相同长度自动分档（阈值 3）：`>=6` 高、`4~5` 中、`3` 低。

## 用法

```bash
# 1) 免费预览 + 落地完整报告（full.json，尚未认证）—— 全程离线
java -jar audit.jar run <你的代码目录> --reference <上游目录> --out-dir ./out

# 2) 取 402 账单（联网，POST 预言机）
java -jar audit.jar request-402 --out ./out/bill.json

# 3) 付款（买家侧 CLI，需先装支付宝 AI 钱包，见下方「前置：买家付款能力」）
alipay-bot 402-buyer-pay --file ./out/bill.json \
  --resource-url "https://pay.seika.ltd/api/audit" --method POST \
  --intent-summary "解锁 Java 代码所有权体检完整报告"

# ⚠ 关键：扫码付款时上一步【不返回 Payment-Proof】，必须跑这条取回执
alipay-bot 402-query-payment-status --trade-no <交易号> \
  --resource-url "https://pay.seika.ltd/api/audit" --method POST --data '{}'

# 4) 离线验签（java.security RSA2）→ 应输出 RECEIPT_VALID
java -jar audit.jar verify --receipt ./out/receipt.json

# 5) 认证完整报告
java -jar audit.jar embed --report ./out/full.json \
  --receipt ./out/receipt.json --out ./out/certified.json
```

> `unlock --proof <Payment-Proof>` 仅适用于能直接拿到 proof 的非扫码付款模式；
> 扫码模式下 proof 由支付宝异步下发，走上面的 `402-query-payment-status` 路径。

## 前置：买家付款能力（`alipay-bot` 从哪来）

付款步骤用到 `alipay-bot` 命令，来自**支付宝官方的 Agent 支付 skill**
（给你的 Agent 一个 AI 钱包，让 AI 帮你下单付款）。**本 skill 不内置、不代管钱包。**

```bash
npx -y @alipay/agent-payment@latest install-experience   # 装完自检：
alipay-bot check-wallet                                  # 看钱包状态
alipay-bot apply-wallet                                  # 首次需绑定自己的支付宝账号
```

> **边界**：`apply-wallet` / `close-wallet` 会改动真实钱包绑定（不可逆），**必须由用户本人确认执行，Agent 不得擅自代跑**。

## 付费认证（x402 回执）

审计本身**完全在客户本机离线跑**，不联网、不出代码。只有「付钱」这一步联网；
付完后服务器签一张**履约回执**，客户端用内置商户公钥**离线**校验，不依赖服务器在线。

架构分工：
- **服务器（a2m-pay X402 预言机）= 纯支付预言机**：只做 402 签发 → 真实验付 → 履约确认 →
  签发 RSA2 签名回执。**不持有、不生成任何审计内容**；付费解锁的是本机已算好的完整报告。
- **本 skill（audit.jar）= 审计执行体**：纯 AST 分析，离线产出 verdict 与完整报告。
- **paygate 子命令 = 支付门禁**：编排付钱这一步 + 离线校验回执真伪 + 把认证块并入报告。

商户应用公钥已**内嵌在 `Paygate.SERVER_PUBKEY_PEM`**（与服务器私钥配对），用于校验回执；
任何字段被篡改都会验签失败。公钥属公开分发物，内嵌不构成安全问题。
如需轮换密钥或指向自建预言机：在同目录放 `server_pubkey.pem`，或用 `--pubkey <路径>` 覆盖。
**免费预览完全不需要网络与上述依赖。**

## 输出

完整报告（`run` 落地的 `full.md` / `full.json`）示例：

```
verdict     : derivative
files       : 1 (1 compared)
literal     : 0 lines
structural  : 13 statements   [high]
threshold   : 3

structural matches (same code, names differ):
    13 stmts  ApiService.java  [high]
         L5: If ... Raise Call Fn:IllegalArgumentException ...
```

免费预览（`preview.md`）只给数量、类型与一句话摘要，**没有行号、没有代码、没有修复建议**。

`verdict` 三种取值：

- `original` — 无 3 行以上连续相同表达代码，且无字节相同文件
- `derivative` — 存在上述情形
- `unknown` — 未提供 `--reference`，或提供了却没有任何文件能对应上（无有效比对）；此时只统计豁免构成，不可当作通过

## 边界

- 只处理 Java（`.java`）。语法错误的文件列在 `unparsable`，不中断整体分析
- 上游缺对应文件时列入 `uncompared`，不当作通过；若提供了 `--reference` 却**没有任何文件能匹配上**（compared=0），判定为 `unknown` 而非 `original`
- 文件配对按相对路径 + 同名进行：演绎方若改了文件名或挪动目录层级，比对会显示 `no reference`，需手动指定对应文件或保持目录结构一致
- 纯本地 AST 分析，不联网、不调用模型
- **审计全程离线**：除「付钱这一步」外，客户代码与报告不出本机、不触网；
  付款后服务器仅回传一张可离线校验的签名回执，不回收任何审计数据
- **这是工程判断，不是法律意见**。结论用于自查和排序，正式场合请咨询律师

## 判定阈值的依据

3 行来自实测：在 363 个净室重写模块上统计，残余相同片段几乎全部落在豁免类别内，真正的表达重合极少超过 2 行。
低于 3 行的相同往往是收敛而非复制。Java 语义（注解/泛型/lambda）与 Python 不同，初期沿用 3，后续用真实净室样本校准。

## 运行环境与权限

| 项目 | 说明 |
|---|---|
| JRE | **≥ 17**（审计引擎 + JavaParser 全烤进 jar，用户侧零装包） |
| 检测缺失 | SKILL 应检测 `java -version`，缺失给安装命令（Arch: `paru -S jdk17-openjdk`；Debian: `apt install openjdk-17-jre`；macOS: `brew install openjdk@17`） |
| 网络访问 | **免费档零网络访问**；仅付费档的付款那一步联网访问支付预言机 |
| 上传数据 | **none** —— 代码不出本机 |
| 模型调用 | **none** —— 不调用任何 LLM |
| 文件写入 | 只写 `--out-dir` 指定的产物目录，不改动源码目录 |

---

## 闭环说明（已定稿）

1. **定价**：**¥0.99 / 次**（高于 Python 版 ¥0.2，已与用户确认）。
2. **预言机 / 密钥**：复用现有 `pay.seika.ltd/api/audit` 同密钥对，按 `resource_id=/api/audit/java`
   路由到支付宝服务 `API_12E012B1842A4F20`（¥0.99）。内嵌 `SERVER_PUBKEY_PEM` 不变。
3. **x402 流程**：`DEFAULT_ORACLE = https://pay.seika.ltd/api/audit`，客户端自动带 `resource_id` 请求；
   付款后预言机返回带 RSA2 签名的回执，客户端用内嵌公钥离线验真后嵌入报告。
