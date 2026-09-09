# 代码所有权体检（Java 版）· Code Ownership Audit for Java

**判定 Java 代码是原创还是演绎作品** —— 给出与上游最长相同表达片段、逐条豁免依据和风险清单。

本仓库是 Python 版 `code-ownership-audit` 的 **Java 移植**：审计引擎用
[JavaParser](https://javaparser.github.io/) 重写，依赖全部 **shade 进 `audit.jar`**，
用户侧**零装包**——只要有一份 **JRE 17+** 就能 `java -jar` 一条命令跑，不需要 Maven / pip。

> ⚠️ **定价与预言机地址为占位/待确认值**，见文末「待确认」区块。最终数额由服务端环境变量驱动。
> ⚠️ **可执行 `audit.jar` 暂未发布**：收费闭环（定价 / 预言机 / x402 真钱流程）待定，当前仓库不含可运行 jar，仅文档先行公开；闭环后推送。

---

## 它解决什么问题

你引入了一段开源 Java 代码，改了改，现在它算谁的？

- **法律上，"改过"不等于"是我的"。** 演绎作品（derivative work）仍受上游许可证约束。
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

## 安装 / 运行前提

| 项目 | 说明 |
|---|---|
| JRE | **≥ 17**（`paru -S jdk17-openjdk` / `apt install openjdk-17-jre` / `brew install openjdk@17`） |
| 装包 | **不需要** —— JavaParser + Jackson 已烤进 `audit.jar` |
| 网络 | 免费档**完全离线**；仅付费档的付款那一步联网 |
| 上传数据 | **none** —— 代码不出本机 |

直接命令行用：

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
| **价格** | **免费、不限次** | **¥0.2 / 次（占位，待确认）** |
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
我们没有加壳也没有混淆 —— **代码保持干净可读**。定价逻辑落在凭证价值上。

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
| 审计引擎（免费档） | **无** —— 已烤进 `audit.jar`（JavaParser + Jackson） |
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

## 待确认（CHECKPOINT）

1. **定价**：Java 版每解锁一次完整报告的价格（文档 §0 要求高于 Python 版 ¥0.2；当前占位 ¥0.2 待你定档）。
2. **预言机 / 密钥**：新 Java 容器复用现有 `pay.seika.ltd/api/audit` 同密钥对 + 新 `RESOURCE_ID`，
   还是另起独立容器 + 新密钥对（届时需更新内嵌 `SERVER_PUBKEY_PEM`）。
3. **x402 流程**：上方 `resource-url` 是否即最终 Java 预言机地址（`DEFAULT_ORACLE` 当前指向
   `https://pay.seika.ltd/api/audit`）。
