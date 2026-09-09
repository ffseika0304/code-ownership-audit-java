# code-ownership-audit · Java 版架构与落地框架

> 文档用途：供「x402 主对话」据此实现 Java 版 code-ownership-audit。
> 主对话已知 x402 全流程，本文档只框定**架构、复用边界、Java 特有差异、验收标准**，
> 不重复 x402 协议细节（见 `POSTMORTEM.md` 与 `a2m_main_new.py`）。
> 参考实现：`_pub/code-ownership-audit/`（Python v1.3.4，已真钱闭环）。

---

## 0. 定位与决策（已拍板，勿改方向）

| 项 | 决策 |
|---|---|
| 形态 | **只做 AI Skill**（SKILL.md 目录结构，audit 用 Java 跑）。只有 skill 形态能走合法商业闭环（DSH / 腾讯 SkillHub / WorkBuddy 市场） |
| 定价 | **高于 Python 版**（Python 现 ¥0.2/次）。具体档位用户另行设定 |
| 预言机 | **独立 X402 容器**：新增一个商品（`RESOURCE_ID` / `GOODS_NAME` / `AMOUNT`），与 Python 现版完全隔离，不动现有 a2m-pay 容器 |
| 企业服务 | 面向开发者/公司的企业级服务是**另一条赛道**，不走这种按次便宜商品，用户本人直接对接，与本次 Java SKill 商品无关 |

---

## 1. 整体架构

```
┌──────────────────────────────────────────────────────────────┐
│  预言机（纯支付 oracle，复用 a2m-pay，零代码改动）              │
│  - 发 402 签名账单 → 真验付 → 履约确认 → 签发 RSA2 回执         │
│  - 不持有任何审计内容；新增一个商品 env 即支持 Java 版           │
│  - Java 用独立容器（独立 RESOURCE_ID / 端口 / 路径）            │
└───────────────────────────┬──────────────────────────────────┘
                             │  HTTP 402 / Payment-Proof（同 Python 协议）
                             │
┌───────────────────────────▼──────────────────────────────────┐
│  Client Skill（纯 Java，离线跑，除"付钱"外不触网）              │
│  ├─ audit.jar ── 静态分析引擎（JavaParser）                     │
│  │     run → preview.md + full.json（未认证）                  │
│  └─ paygate（同一 jar 的子命令）                               │
│        request-402 → verify(RSA2, java.security) → embed       │
└──────────────────────────────────────────────────────────────┘

数据流：本地离线算 full.json → 取 402 账单 → 付款 → 回执 → 离线验签 → certified.json
```

**关键不变式**：审计永远在客户本机离线完成，服务器只做支付仲裁。付费解锁的是"带签名回执的认证交付物"，不是计算结果。

---

## 2. 复用清单（从 Python 版直接搬，勿重造）

| 复用项 | 来源 | 说明 |
|---|---|---|
| 预言机服务端 | `a2m_main_new.py` / `engine.py` | 仅新增一个商品 env，**代码零改动** |
| 402 协议字段 | `paygate.py` | `Payment-Needed` 头 / `X-Out-Trade-No` / `Payment-Proof` 头 / 回执 JSON 结构 / canonical 串算法 |
| freemium 分层模型 | `audit.py:build_preview` | preview 剥离行号/代码位置/修复建议 |
| SKILL.md 文案结构 | `SKILL.md` | 判定方法、阈值依据、边界条款 |
| 公钥嵌入策略 | `paygate.py:SERVER_PUBKEY_PEM` | 内嵌常量 + 同目录 `server_pubkey.pem` 覆盖入口 |
| 12 个踩坑复盘 | `dist/POSTMORTEM.md` | 实现 + 上架时逐条核对 |

---

## 3. 目录结构（Java skill 成品）

```
_pub/code-ownership-audit-java/
  SKILL.md                 # Java 版元数据 + JRE 检测 + 用法
  dsh-plugin.json          # 上架元数据（slug 区分 java 版）
  audit.jar                # fat jar：含 JavaParser + 审计引擎 + paygate 编排
  server_pubkey.pem        # 可选；不随包发也可（内嵌常量已够）
  README.md
  src/test/resources/fixtures/   # JUnit 测试样本（原创/演绎/字节相同）
```

**用户侧只需**：`java -jar audit.jar <args>`，无需 maven / pip。

---

## 4. 依赖模型（最关键）

- Java **无 stdlib 源码级 AST** → 必须引 **JavaParser**（`com.github.javaparser:javaparser-core`，MIT）。
- 发布做成 **fat / uber jar**（maven-shade-plugin 或 gradle shadowJar），把 JavaParser 全部 shade 进 jar。
- **用户侧零包管理器**：只 `java -jar`，不需要 `mvn` / `pip` 装任何库。
- 唯一前提 = **JRE 17+**；`SKILL.md` 必须检测 `java -version`，缺失给安装命令
  （Arch: `paru -S jdk17-openjdk`；Debian: `apt install openjdk-17-jre`；macOS: `brew install openjdk@17`）。
- HTTP 用 `java.net.HttpURLConnection`（stdlib，不引额外依赖）。
- RSA2 验签用 `java.security`（stdlib，`Signature "SHA256withRSA"`）。
- ⚠ **"零依赖"卖点 Java 站不住**（JavaParser 对免费档也不可避免），但用户体验与 Python 等价（一个 jar、一条命令）。文案改用"freemium 代码层剥离"，不再声称零依赖。

> 对比 Python：Python 付费档还要 `pip install pycryptodome`；Java 把依赖烤进 jar，用户侧反而更省一步。代价是 JRE 普及度低于 Python 解释器，故 SKILL.md 检测不可省。

---

## 5. 付费闭环流程（Java 版等效 paygate）

```bash
# 1) 离线跑审计：出预览 + full.json（未认证）
java -jar audit.jar run <目录> --reference <上游> --out-dir ./out

# 2) 取 402 账单（联网，POST 预言机）
java -jar audit.jar request-402 --out ./out/bill.json

# 3) 付款（买家侧 CLI，与 Python 版完全相同，本 skill 不内置钱包）
alipay-bot 402-buyer-pay --file ./out/bill.json \
  --resource-url "https://<java-oracle>/api/audit" --method POST --intent-summary "解锁 Java 代码所有权体检完整报告"

# 4) 扫码付款时 proof 不在第 3 步返回，必须跑这条取回执
alipay-bot 402-query-payment-status --trade-no <交易号> \
  --resource-url "https://<java-oracle>/api/audit" --method POST --data '{}'

# 5) 离线验签（java.security RSA2）→ 应输出 RECEIPT_VALID
java -jar audit.jar verify --receipt ./out/receipt.json

# 6) 认证完整报告
java -jar audit.jar embed --report ./out/full.json \
  --receipt ./out/receipt.json --out ./out/certified.json
```

> `alipay-bot` 付款流程完全复用，客户端只把"验签/编排"从 Python 换成 Java。

---

## 6. 验签实现（必须与预言机逐字节一致）

- **canonical 串算法**（与 `paygate.py:_canonical` 完全一致，Java 端逐字节复刻）：
  ```
  canonical = "&".join("k=v" for k in sorted(receipt.keySet()))
  ```
  字段按字典序、`key=value`、用 `&` 连接，**不含任何空格/换行**。
- **SHA256 → RSA2 (PKCS#1 v1.5)** 验签：`java.security.Signature.getInstance("SHA256withRSA")`，
  用商户公钥 verify。
- **公钥处理**：
  - 内嵌常量 `SERVER_PUBKEY_PEM`（与 Python 版同一个 PEM，若新容器**复用同密钥对**）。
  - 支持同目录 `server_pubkey.pem` 覆盖（密钥轮换 / 自建预言机时）。
  - ⚠ **若新 Java 容器用新密钥对**：构建后导出其公钥，嵌入 Java 常量，或放 `server_pubkey.pem`。Java 内嵌公钥必须匹配新容器签名私钥，否则验签全失败。
- 任何字段（含 `fulfilled_at`）被篡改 → 验签失败。

---

## 7. 审计引擎（audit.jar 核心，从 `audit.py` 移植）

两层测量模型（与 Python 同构）：

1. **literal（字面层）**：逐行比对相同源码行，剔除"语言强制 / 公开接口"行。
2. **structural（结构层）**：AST 形状比对，局部变量名归一化为槽位（`v0/v1/...`），调用目标与属性名保留。

**Java 版豁免类别**（对应 Python 的 8 类，按 Java 语法映射）：

| 类别 | Java 示例 | 为什么豁免 |
|---|---|---|
| `import` / `package` | `import java.util.*` | 名字由依赖/包结构决定 |
| `signature` | `public int run(String x)` | 公开接口，改名破坏调用方 |
| `annotation` | `@Override` / `@Service` | 绑定框架定义 |
| `control_flow` | `for (...)` / `if (...)` | 控制流只有少数写法 |
| `keyword` | `return` / `throw` / `break` | 语言语法 |
| `type_decl` | `class` / `interface` / `enum` | 公开类型契约 |
| `field_binding` | `this.name = name` | 构造器存字段的惯用写法 |
| `assert` / 字面量常量 | `assert x == 1` | 陈述公开契约 |

- **字节相同文件 → derivative**（防止表达行被签名行分隔而漏判）。
- **阈值**：默认 `3`（连续相同表达行即判演绎），天花板 `8`（参考 Python `THRESHOLD_CEILING`，高于此拒绝——无观测支撑）。Java 人群基线（363 模块式测量）后续实测可校准，初期沿用 3。
- **verdict**：`original` / `derivative` / `unknown`（同 Python 边界：`--reference` 未匹配到任何文件 → `unknown`，不可作通过依据）。
- **`buildPreview(report)`**：只返回 数量 / 类型分布 / 一句话摘要，**剔除行号、代码位置、修复建议**（freemium 代码层剥离，防止从预览反推完整内容）。
- 输出：`renderMarkdown(full)` / `renderPreviewText(prev)` 等效 Python。

---

## 8. 价格驱动

- 权威价在服务端 env `AIPAY_AMOUNT`（Java 新容器），客户端 `PRICE_CNY` 仅离线提示兜底。
- 真实金额以 **402 账单 + 回执签名校验**为准（防篡改），不受客户端常量影响。
- Java 定价档位：用户另行设定（高于 Python）。

---

## 9. 构建步骤（作者侧一次性）

1. `pom.xml`：依赖 `javaparser-core` + `maven-shade-plugin`（shade 进 fat jar）。
2. 包结构：
   - `engine/` — 审计引擎（literal / structural / buildPreview / render）
   - `paygate/` — `request402()` / `verifyReceipt()` / `embed()` / CLI `main`
   - `model/` — `Report` / `Finding` / `Preview` POJO（对应 Python dict）
3. **JUnit 覆盖五类**（对应 Python 31 项 pytest）：literal / structural / preview 剥离 / verify 验签 / embed 认证。
4. 打包约束：
   - **不含 `.pem` 附件**（腾讯 SkillHub 拒收）→ 公钥内嵌源码常量。
   - 体积参考 Python 58514B / 7 文件；Java jar 因含 JavaParser 会大些（正常）。

---

## 10. 验收（真钱闭环自检，必做）

- [ ] 本地 `run` + `buildPreview` 出免费预览（无行号/无修复）。
- [ ] 真实走一遍 `402 → 付款 → query → verify → embed`，`verify` 输出 `RECEIPT_VALID`。
- [ ] 篡改回执任一字段 → `verify` 失败（验签正确拒绝）。
- [ ] 三个渠道（DSH / 腾讯 SkillHub / WorkBuddy 市场）按 Python 上架流程复用；`dsh-plugin.json` slug 区分 java 版。
- [ ] 对照 `dist/POSTMORTEM.md` 12 坑逐条核对（尤其卖家号、Request 名称遮蔽、扫码 proof 异步、`.pem` 拦截）。

---

## 11. 与 Python 版差异清单（勿盲目照搬）

| 维度 | Python 版 | Java 版 |
|---|---|---|
| 免费档依赖 | stdlib `ast`，真·零依赖 | 必带 JavaParser（烤进 jar） |
| 运行前提 | 需 `pycryptodome`（付费档） | 需 JRE 17+（全档） |
| 发布物 | `.py` 源码（system python 直跑） | `.jar`（预编译，用户 `java -jar`） |
| AST 库 | `ast` | JavaParser |
| 验签库 | `pycryptodome` (RSA/PKCS1_v1_5) | `java.security` (`SHA256withRSA`) |
| 包管理负担 | 落在用户侧（付费档 pip） | 落在作者侧（构建时 shade），用户侧零 |

---

## 12. 风险 / 坑

1. **密钥对应**：Java 内嵌公钥必须匹配新 Java 容器签名私钥（复用同密钥对最省事）。
2. **JRE 普及度**：agent 沙箱可能无 JRE，`SKILL.md` 检测 + 给安装命令不可省。
3. **canonical 串逐字节一致**：排序、连接符、空格任一偏差 → 验签全失败。
4. **env 隔离**：新商品 `RESOURCE_ID` 不与 Python 现商品冲突；两容器独立运行。
5. **阈值基线**：Java 语义（注解/泛型/lambda）与 Python 不同，先沿用 3，后续用真实净室样本校准，避免误判率漂移。

---

## 13. 一句话给实现者

> 把 `_pub/code-ownership-audit/` 的 `audit.py` + `paygate.py` 用 Java 重写一遍，
> 依赖用 fat jar 藏起来，验签换成 `java.security` 且 canonical 串与预言机逐字节对齐，
> 预言机只加一个商品 env 即可。freemium 剥离规则、402 协议、公钥嵌入策略全照搬。

---

## 14. 实现环境：在本机 WSL Arch 中完成（勿在 Windows 侧留临时文件）

**硬约束**：所有源码编写、编译、fat jar 打包、真钱闭环自检，**一律在 WSL Arch 里跑**，
不要在 Windows 桌面 / Downloads / 用户 Temp 丢 `.java`、`.class`、`.jar` 半成品或 `target/` 目录。
WSL 的 `/mnt/d` `/mnt/e` 已直挂 Windows 盘，产物要落到 Windows 侧时直接写 `/mnt/d/...`，
不要 `cp` 回桌面再清理——从源头就别放 Windows 本地目录。

### 14.1 入口（实现者用 WorkBuddy Bash 即 Git Bash，前缀调起 WSL）

```bash
wsl -d Arch                       # 以默认用户 seika 进交互（发行版当前 Stopped，此命令会自动启动）
wsl -d Arch -u root -- <cmd>      # 以 root 执行单条命令（装包/起服务用，无密码）
wsl -d Arch -- <cmd>              # 以 seika 执行单条命令
```
- 发行版名 `Arch`（已设为默认）。用户 `seika`（密码 `0304`），`root` 经 `wsl -u root` 直接提权不弹密码。
- systemd 在 WSL 内正常运行（`systemctl is-system-running → running`），docker/服务按 systemd 单元管。

### 14.2 WSL Arch 已就绪的能力（直接复用，别重装）

| 能力 | 状态 | 备注 |
|---|---|---|
| 系统 | Arch Linux，systemd | — |
| 磁盘挂载 | `/mnt/c` `/mnt/d` `/mnt/e` `/mnt/wsl` | 项目放 `/mnt/d/wsl/` 下可跨 WSL 重装存活 |
| 网络 | mirrored 直连外网 + DE 加速透传 | 能直连阿里云 `39.106.167.67`（a2m-pay 预言机）做闭环自检 |
| docker 29.7.2 | ✅ 已配 | registry-mirror → `189.24.83.138:5000`（mTLS 经 DE），pull 已验证 |
| paru v2.1.0（AUR） | ✅ | `paru -S xxx` 直装，不用手动 clone |
| git 2.55.0 | ✅ | github 已系统级 `insteadOf` → DE:8443，拉 SourceCodeForge/gitee 不卡 |
| python3 3.14.7 / gcc 16.2.1 | ✅ | 仅备用 |
| node / go | ❌ 未装 | 本任务不需要；要用 `paru`/`pacman` 补 |

### 14.3 实现前必装（一条命令）

```bash
wsl -d Arch -u root -- bash -c 'pacman -Sy --noconfirm jdk17-openjdk maven'
# 验证：
wsl -d Arch -u root -- bash -c 'java -version; javac -version; mvn -version'
```
- JDK 17 提供 `java` / `javac`；Maven 用于 shade 出 fat jar（若要 gradle 改 `pacman -S gradle`）。
- JavaParser 走 Maven 依赖引入（`com.github.javaparser:javaparser-core`），shade 进 jar，用户侧零安装。
- **Maven 拉中央仓库**：WSL 直连外网可直拉 Maven Central；若想加速，在 WSL 内加 Aliyun 镜像
  `~/.m2/settings.xml`：
  ```xml
  <mirror><id>aliyun</id><mirrorOf>*</mirrorOf>
    <url>https://maven.aliyun.com/repository/public</url></mirror>
  ```
  （注意：git/docker 已走 DE 代理，**Maven 不在 DE 代理范围内**，用 Aliyun 镜像或直连即可。）

### 14.4 项目落地路径（推荐）

```
/mnt/d/wsl/x402-java/            # 工作根（在 Windows D: 上，WSL 重装不丢）
  src/  pom.xml  target/         # 源码与构建产物（留在 WSL/D 盘，不进 Windows 用户目录）
  dist/audit.jar                 # 最终 fat jar
```
- Python 参考实现 `_pub/code-ownership-audit/` 在 Windows 会话目录，**进 WSL 后通过 `/mnt/d/...` 读**，
  不要 `cp` 到桌面再读——直接 `cat /mnt/d/Users/82760/WorkBuddy/2026-08-07-14-46-56/_pub/code-ownership-audit/*.py`。
- 成品 skill 目录：`/mnt/d/Users/82760/WorkBuddy/2026-08-07-14-46-56/_pub/code-ownership-audit-java/`
  （WSL 内路径即上面那串，已挂 `/mnt/d`），`audit.jar` 直接写进这个目录。

### 14.5 闭环自检时的网络

- 预言机在阿里云 `39.106.167.67`（a2m-pay 容器），WSL 经家庭网络直连可达 → `request-402` / `query` 能真跑。
- 付款用 `alipay-bot`（与 Python 版同流程），客户端只在 `verify` 阶段离线验签，不依赖外网。
- 若 Java 新容器另起端口/路径，把 oracle endpoint 写进 `paygate` 常量即可，与 WSL 无关。

### 14.6 收尾（交付后保持本机干净）

- 构建产物只在 `/mnt/d/wsl/x402-java/` 与 `_pub/code-ownership-audit-java/` 两处；
  **不在** `C:\Users\82760\Desktop` / `Downloads` / `%TEMP%` 留任何 `.java` `.class` `target/` `*.jar` 半成品。
- 实现完若想释放空间：`wsl -d Arch -u root -- rm -rf /mnt/d/wsl/x402-java/target`（保留 `dist/audit.jar` 源）。
- 配置脚本 `D:\wsl\de-proxy-wsl.sh` / `de-proxy-wsl2.sh` 已把 git/docker/pacman 路由到 DE/清华，
  重装 WSL 后重新执行即可，无需重新研究（证书需从 DE 重新拉，勿留私钥副本在 Windows 侧）。
