# 玄机量化 XuanJiQuant 功能说明与操作使用手册

> 版本：v1.0.0　｜　适用系统：Windows 单机
> 最后更新：2026-09-26
> 阅读对象：第一次使用本软件、不熟悉系统结构的操作员 / 研究员 / 风险审阅人员

---

## 目录

1. [这个软件是做什么的](#1-这个软件是做什么的)
2. [必须先知道的 6 条安全边界](#2-必须先知道的-6-条安全边界)
3. [安装与环境要求（首次部署）](#3-安装与环境要求首次部署)
4. [启动软件与本地身份登录](#4-启动软件与本地身份登录)
5. [界面总览与角色权限](#5-界面总览与角色权限)
6. [各功能页面详细说明](#6-各功能页面详细说明)
   - 6.1 [投资驾驶舱（首页总览）](#61-投资驾驶舱首页总览)
   - 6.2 [因子研究](#62-因子研究)
   - 6.3 [策略运行（F4 验证与研究选股）](#63-策略运行f4-验证与研究选股)
   - 6.4 [Qlib 机器学习实验](#64-qlib-机器学习实验)
   - 6.5 [模拟执行（F5 运行监控）](#65-模拟执行f5-运行监控)
   - 6.6 [模拟组合（资金与持仓）](#66-模拟组合资金与持仓)
   - 6.7 [风险监控（风险 / 诊断 / 审计回放）](#67-风险监控风险--诊断--审计回放)
   - 6.8 [告警中心](#68-告警中心)
   - 6.9 [市场数据（行情 / 估值 / 财务 / 数据管理）](#69-市场数据行情--估值--财务--数据管理)
   - 6.10 [市场资讯（金十 / 巨潮）](#610-市场资讯金十--巨潮)
7. [日常使用流程（建议照做）](#7-日常使用流程建议照做)
8. [命令行运维手册（进阶）](#8-命令行运维手册进阶)
9. [核心概念通俗解释](#9-核心概念通俗解释)
10. [数据文件与目录说明](#10-数据文件与目录说明)
11. [权限模型与 API 令牌配置](#11-权限模型与-api-令牌配置)
12. [常见问题 FAQ](#12-常见问题-faq)
13. [附录：API 端点 / 计划任务 / 配置文件速查](#13-附录api-端点--计划任务--配置文件速查)

---

## 1. 这个软件是做什么的

玄机量化是一套**面向 A 股的本机量化研究 + 确定性模拟盘工作台**，把量化投研的整条链路放在一台电脑上完成：

```text
数据接入（行情/财务/资讯）
  → 数据质量门禁（覆盖率、PIT 防未来函数）
  → 因子研究（58 个登记因子 + IC/IR 评估）
  → 策略验证（F4 滚动走样本外，诚实检验能否跑赢指数）
  → Qlib 机器学习实验（LightGBM / XGBoost / 线性模型）
  → 每日研究选股（research_only，仅研究用途）
  → F5 模拟成交（独立 100 万模拟账本，T+1、费用、滑点、涨跌停全部仿真）
  → 硬风控与审计（逐单规则、十项对账、全程留痕）
  → Web 界面查看（只读为主，写入受令牌保护）
```

一句话理解：**它帮你验证"一个选股想法在真实交易成本下到底灵不灵"，并且只在模拟账本里跑，永远不会真的下单花钱。**

### 它不做什么

- 不连接券商、不碰真实资金，**永远不能实盘交易**；
- 没有"AI 自动炒股"：AI 自治控制面已于 2026-08-13 退役，AI 仅保留**只读的影子研究/解读**功能（如快讯解读、个股证据摘要），输出没有任何执行效力；
- 不承诺收益。回测和模拟盈利不代表未来。

---

## 2. 必须先知道的 6 条安全边界

| # | 边界 | 含义 |
|---|---|---|
| 1 | 实盘永久关闭 | `live_execution_authority=false` 是固定事实，任何下单到券商的动作都不存在 |
| 2 | 研究 ≠ 可交易 | 所有研究产物固定标注 `research_only`、`execution_authority=false`，页面上的选股组合**不构成交易信号** |
| 3 | F4 不过线就不许"转正" | 策略只有在 5 道业绩门槛全过才是 `f4_research_candidate`；不过就是 `f4_rejected`，系统不会替它找借口 |
| 4 | 失败关闭（fail-closed） | 数据缺失、版本对不上、非交易时段等任何不确定情况，系统一律停下并给原因码，绝不"猜一个能跑的结果" |
| 5 | 写操作需要令牌 | 浏览器里大部分"运行/更新/确认"按钮属于控制面动作，需配置 `XUANJI_API_TOKEN` 才能点；纯浏览不需要 |
| 6 | 唯一活动账本 | 全系统只有一个 F5 模拟账本 `data/paper/f5_ledger.db`（初始 100 万现金）；历史账本只读，不参与当前账户 |

---

## 3. 安装与环境要求（首次部署）

### 3.1 环境要求

- **操作系统**：Windows 10/11（64 位），使用 PowerShell 或 CMD
- **Node.js**：18 或更高（前端 + API 服务）
- **Python**：3.10+（量化核心）；Qlib 训练建议使用项目自带的独立虚拟环境 `.venv-qlib`（Python 3.11 + pyqlib 0.9.7）
- **硬件建议**：16GB 以上内存、多核 CPU；Qlib/F4 全量训练是 CPU 密集型任务
- **网络**：能访问免费数据源（通达信 TdxQuant、新浪/腾讯行情、AkShare、BaoStock、巨潮、金十），免费源可能限流或偶发失败

### 3.2 首次安装步骤

在项目根目录 `d:\XuanJiQuant-v1.0.0-portable` 打开 PowerShell：

```powershell
# 1) 安装前端依赖
npm install

# 2) 安装 Python 量化核心依赖
pip install -r requirements.txt

# 3) （需要机器学习实验时）安装 qlib 隔离环境
python -m venv .venv-qlib
.\.venv-qlib\Scripts\python.exe -m pip install -r requirements-qlib.txt
```

### 3.3 数据初始化（新机器 / 空库）

界面和服务不依赖全量数据即可启动，但要做研究需要先有数据，按由轻到重的顺序：

| 步骤 | 命令（Python 用默认解释器即可） | 耗时参考 | 说明 |
|---|---|---|---|
| 日线行情 | `python scripts\daily_update.py` | 约 30 分钟 | 全 A 股增量日 K，收盘后运行 |
| 财务数据 | `python scripts\daily_update.py --financial` | 约 5 小时 | 季度任务，不用天天跑 |
| 因子评估 | `python scripts\evaluate_factors.py` | 约 30 分钟 | 生成 58 因子有效性评估 |
| 六年 PIT | `.venv-qlib\Scripts\python.exe scripts\collect_six_year_pit.py` | 数小时 | Qlib/F4 训练用的六年历史数据 |

> 也可以在网页内操作：**市场数据 → 数据管理**（增量更新 K 线 / 刷新财务数据，需令牌），以及 **Qlib 实验 → 数据准备**（六年数据"检查目录→补齐→质量门禁→导出 bin"四步向导，需令牌）。

数据是否齐全，可在网页 **市场数据 → 数据管理** 查看股票数、K 线数量、最新交易日、数据源健康度。

---

## 4. 启动软件与本地身份登录

### 4.1 启动（推荐方式）

双击项目根目录的 **`start_all.bat`**，它会：

1. 检查 `data\quant.db` 是否存在；
2. 调用 `node scripts\start_services.mjs` **幂等启动**两个服务（已在运行的不会重复启动）；
3. 自动打开浏览器。

也可以命令行启动：

```powershell
node scripts\start_services.mjs
```

| 服务 | 地址 | 作用 |
|---|---|---|
| 前端 Web | <http://127.0.0.1:8888> | 浏览器操作界面 |
| 后端 API | <http://127.0.0.1:8880> | 数据接口，负责拉起 Python 量化进程 |

两个地址都**只监听本机 127.0.0.1**，局域网其他电脑访问不到，这是安全设计。

> 后端通过环境变量 `PYTHON` 决定调用哪个 Python。若使用 qlib 虚拟环境启动后端，需先设：
> PowerShell：`$env:PYTHON='d:\XuanJiQuant-v1.0.0-portable\.venv-qlib\Scripts\python.exe'`

### 4.2 登录：选择一个本地身份

打开网页后首先看到身份选择页。这不是真正的账号密码系统，只是本机的**角色视角切换**（选择结果存在浏览器 localStorage），三个角色看到的菜单不同：

| 身份 | 适合谁 | 能看到的页面 |
|---|---|---|
| **模拟盘操作员** | 想用全部功能的人（推荐首次使用选这个） | 全部 10 个页面 |
| **风险审阅员** | 只关心风险/告警/成交审计 | 驾驶舱、模拟执行、模拟组合、风险、告警、市场数据、市场资讯 |
| **策略研究员** | 专注研究、不看模拟交易 | 驾驶舱、市场数据、因子、策略、风险、告警、资讯、Qlib |

左下角可随时**退出身份**重新选择。

### 4.3 关闭软件

直接关浏览器只会关掉页面，后台两个服务仍在运行。停止服务可关闭对应的终端窗口，或在任务管理器结束 node 进程；用计划任务托管时见 [第 8 节](#8-命令行运维手册进阶)。

---

## 5. 界面总览与角色权限

登录后界面分为三块：

- **左侧导航栏**：五个功能分组（可折叠）；
- **顶部指数条**：每 15 秒刷新主要指数（上证、深证、创业板等）实时点位和涨跌幅，红涨绿跌；
- **主内容区**：当前页面。

```text
决策中心    投资驾驶舱
策略研究    因子研究 · 策略运行 · Qlib 实验
交易与组合  模拟执行 · 模拟组合
风险与审计  风险监控 · 告警中心
数据与系统  市场数据 · 市场资讯
```

每个研究类页面顶部都有一条 **ResearchBoundary 研究边界条**，明确写出"数据截止日 / 当前选股日 / 不构成交易信号"，这是合规要求，不是错误提示。

---

## 6. 各功能页面详细说明

### 6.1 投资驾驶舱（首页总览）

**入口**：决策中心 → 投资驾驶舱。
**用途**：一屏看清"钱、风险、数据新鲜度、研究组合"。

页面内容（全部只读，右上角"刷新"可手动更新）：

| 区域 | 展示内容 |
|---|---|
| 核心指标卡 | 总权益、今日盈亏、综合风险（低/中/高）、持仓数量 |
| 数据新鲜度 | 行情时间、权益估值时间、风险计算时间、数据状态（新鲜/过期） |
| 影子研究状态 | AI 只读影子研究的对照报告、影子记录库状态（无基线产物时显示跳过，属正常） |
| 研究目标组合 | 每日研究选股的代码、名称、**目标权重**、研究理由——注意这是 research_only 研究组合，不是下单指令 |

**怎么用**：每次打开软件先看这一页。若"数据状态"显示过期，先去市场数据页更新；若想看钱和持仓去模拟组合页。

---

### 6.2 因子研究

**入口**：策略研究 → 因子研究。共 3 个子标签：

#### ① 市场榜单
- 全市场个股按因子综合得分的排行榜，调用 `market_eval`；
- 数据来自每日流水线发布的最新因子快照（58 个因子）；
- 加载失败时可点"重试"。若长期失败，通常是当天因子快照未生成（先跑每日流水线，见第 7、8 节）。

#### ② 因子列表
- 展示全部登记因子：名称、中文名、分类（技术面/基本面/价量等）、含义说明；
- 点击某个因子可查看其 **Top15 / Bottom15 成分股**（该因子打分最高和最低的各 15 只）。

#### ③ 批量 IC
- IC（信息系数）= 因子值与未来涨幅的相关性，用来量化"因子有没有预测力"；
- 操作：在输入框填入股票代码（逗号分隔，如 `600519, 000001, 300442`），选择因子；
- 支持单股因子值评估、分段评估（自动计算未来 1/5/10/20 日收益的相关性）、以及对一批股票全部因子批量评估；
- **注意**：这些"评估/计算"按钮是控制面动作，未配置令牌时会提示 403（见第 11 节）。日常研究只需看榜单和列表即可，无需手动算。

---

### 6.3 策略运行（F4 验证与研究选股）

**入口**：策略研究 → 策略运行。共 4 个子标签，页面同时并列展示两条独立证据链：

- **F4 / PIT 证据**（周频更新，看"数据截止日"）：策略走样本外验证状态；
- **每日研究选股**（日频更新，看"当前因子/选股日"）：当天生成的研究组合。
- 任一失败都不会覆盖另一条。

#### ① 诊断：多因子
- 选若干个因子、配权重、设持有天数、给股票池，运行 `multi_factor` 多因子组合诊断回测；可删除已选因子（至少留 2 个）。

#### ② 策略列表
- 系统内置策略的清单、参数说明和默认值，例如"单因子排名"（按因子 Z-Score 排序选股）等；
- 同时展示上面说的 F4 周频状态与每日研究选股快照。

#### ③ 诊断：策略回测
- 选一个策略 → 设置参数 → 给股票代码列表 → 运行回测，查看收益表现；
- 这是**诊断工具**，结果不经过 F4 门禁，不能用于模拟盘准入。

#### ④ 历史模拟配置
- 旧版模拟盘的策略参数配置，仅历史留存。

**如何读懂 F4 状态（重要）**：

| 状态 | 含义 | 对 F5 模拟盘的影响 |
|---|---|---|
| `f4_blocked` | 数据/身份门禁没过，证据不完整 | 模拟盘 blocked，零订单（异常状态，需查数据） |
| `f4_rejected` | 数据合规、但 5 道业绩门槛没过（策略不灵） | 可走 experimental 实验模拟车道；策略质量标"不合格" |
| `f4_research_candidate` | 业绩门槛全部通过 | 可进入 validated 正式模拟车道 |

无论哪种状态，研究结论都是 `research_only`，页面上**没有任何"晋升/交易"按钮**。

> 背景知识：F4 用 504 天训练 / 126 天验证选冠军 / 126 天测试的滚动窗口，配合 20 天 purge 和 5 天 embargo 防止未来函数；24 个预登记候选（动量/反转/防御/流动性/组合/Qlib 各 4 个）每个窗口只有验证段第 1 名能进入测试，并按 1.0/1.5/2.0 三档交易成本考核。详见第 9 节。

---

### 6.4 Qlib 机器学习实验

**入口**：策略研究 → Qlib 实验。共 7 个子标签：

| 子页 | 用途 |
|---|---|
| **研究总览** | qlib 环境状态（Python/qlib/LightGBM/XGBoost 版本、ready 状态）、数据集与任务汇总 |
| **数据准备** | 四步向导（按钮依次解锁）：①检查目录 → ②补齐六年原始行情 → ③运行数据质量门禁 → ④导出六年 Qlib bin（质量门禁通过后才可点） |
| **模型训练** | 三个**固定**训练工作流（不接受自定义模型 YAML）：**周度固定基线**（Alpha158+LightGBM）、**Walk-Forward 六年基线**（月度滚动窗口）、**季度固定矩阵**（Alpha158/Alpha360 × LGBM/XGB/Linear 串行矩阵） |
| **实验记录** | 每次 workflow 的 Recorder、参数、指标记录 |
| **模型仓库** | 训练产出的模型列表，达标的可"人工晋升影子"（仅影子信号，非实盘） |
| **回测评估** | qlib 官方回测与 A 股本地化回测双引擎结果 |
| **任务日志** | 活动任务实时日志，可"取消活动任务" |

**特点**：

- 训练窗口严格按 PIT（时点）隔离，训练时看不到未来数据；
- 单个候选失败只标记不可用原因，不会拖垮整条流水线；
- 首次全量训练耗时较长（数小时级，取决于机器），数据缓存（HDF5）建立后同窗复跑会明显加快；
- 训练/导出/晋升/取消均为控制面按钮，需令牌；总览、记录、模型、回测、日志查看均为只读。

详细运维说明见项目文档 [QLIB_LOCAL_TRAINING.md](QLIB_LOCAL_TRAINING.md)。

---

### 6.5 模拟执行（F5 运行监控）

**入口**：交易与组合 → 模拟执行。
**用途**：查看 F5 确定性模拟执行的**运行实况**。本页**全部只读，没有任何手动下单按钮**——F5 只按计划任务在固定时间自动运行，UI/API 都不能提交代码、方向、数量、价格。

页面从上到下：

1. **状态指标区（约 20 项）**：
   - 本轮运行状态、本轮订单数、本轮成交数、当前持仓、权益、现金、市值；
   - 当日/累计盈亏、累计收益率、浮动/已实现盈亏；
   - 组合 ID、验证 ID、目标交易日；
   - **熔断开关**（异常时系统自动拉开）、**执行通道**（validated_paper 正式 / experimental_paper 实验）、**策略质量**（合格/不合格）、**模拟执行许可**、**实盘权限（永远显示未启用）**、执行模式（日频/盘中）、实时行情时间。
2. **当前持仓管理（只读）表**：名称代码、数量、**可卖（T+1）**、当日买入、成本价、现价、市值、浮动盈亏、仓位占比等。
3. **本轮模拟订单表**：方向、委托量、成交/取消量、成交价、状态、**拒绝原因**（如非交易时段、涨跌停、资金不足等都会明确写出）。
4. **本轮模拟成交与对账结果表**：每笔成交（含手续费+印花税）+ 十项不变量对账逐条"通过/失败"。

**F5 什么时候自己跑**（计划任务，见第 8 节）：

- **日频**：交易日 17:10 为"下一交易日"准备模拟计划（卖出优先、买入整手 100 股）；
- **盘中**：交易日 09:35–11:25、13:05–14:50 每 5 分钟用实时行情模拟成交；
- 撚则：次日开盘价 + 滑点 + 佣金/印花税/过户费 + ADV 10% 成交量参与上限 + 停牌/涨跌停/T+1 限制；
- 每轮跑完做十项对账：全部通过才算 completed；若副作用无法证明则 `halted_unknown` 并熔断。

**为什么今天显示 `non_trading_weekday`**：周末或节假日休市，系统正确识别为非交易日，不生成订单，这是正常的。

---

### 6.6 模拟组合（资金与持仓）

**入口**：交易与组合 → 模拟组合。
**用途**：全系统**唯一活动模拟账本**的资产视图，比模拟执行页更聚焦"账户本身"。

展示：总权益、可用现金、持仓市值、当日盈亏、累计盈亏与收益率、浮动/已实现盈亏；持仓明细含 **T+1 可卖数量**（A股当天买入次日才能卖）。

- 初始资金 100 万（配置见 `config/f5_paper_execution.json`）；
- 费用参数：佣金万 3（最低 5 元）、印花税卖出千 0.5、过户费十万分之一、滑点万 1、整手 100 股；
- 本页只读，不能手动改持仓或入金。

---

### 6.7 风险监控（风险 / 诊断 / 审计回放）

**入口**：风险与审计 → 风险监控。3 个子标签：

| 子页 | 内容 |
|---|---|
| **组合风险** | 当前模拟组合的确定性风险指标：仓位、行业集中度、单票敞口等（硬限制定义在 `quant/risk/hard_limits.json`） |
| **系统诊断** | 四层健康度检查（数据源、行情、风控、账本等组件存活与新鲜度），任何一项异常都直接显示失败，不会被粉饰为正常 |
| **审计回放** | 最近 20 次运行记录，点开可对某次决策做逐步回放（每个 run_id/decision_id 的事实链条），用于事后核查 |

原则：**风控失败即关闭，不允许在线放宽规则。**

---

### 6.8 告警中心

**入口**：风险与审计 → 告警中心。2 个子标签：

- **告警记录**：分"业务告警"和"测试/演练"两类筛选；每条告警可**确认（acknowledge）**或**解决（resolve）**；
- **告警规则**：查看全部规则，可启用/停用、静默 1 小时。

**处置告警必须留痕**：点击确认/解决/停用/静默时，弹窗强制填写两项：

1. **操作员**（姓名或代号）；
2. **处理原因**（可复核的处置依据）。

两者都填了才能"确认提交"。这些是控制面动作，需令牌。

---

### 6.9 市场数据（行情 / 估值 / 财务 / 数据管理）

**入口**：数据与系统 → 市场数据。5 个子标签，是日常用得最多的页面。

#### ① 市场浏览
- **成交额前 100 榜单**：代码、名称、最新价、涨跌额/幅、振幅、换手率、主力净流入/净占比、成交量/额（行情主源 TdxQuant，资金流字段由东方财富补充，腾讯仅补换手率）；
- 支持按**代码 / 名称 / 拼音首字母**搜索；
- 点股票可看：**日 K / 周 K / 月 K**，**不复权 / 前复权 / 后复权**切换；
- **逐笔成交**：查看当日逐笔（最多 300 条），可"补齐今日逐笔"做全日回填；
- **自选股**：加自选、移出自选、恢复默认自选列表；
- **导出 CSV**：Top100 可一键导出，Excel 直接打开；
- **AI 个股证据**：对选中股票生成只读证据摘要（AI 无任何操作权限）。

#### ② 股票估值
- 三类估值视角：**绝对估值**（DCF 类）、**相对估值**（PE/PB 等可比口径）、**市场估值**；
- 输入 6 位代码（如 `600519`）分析，可强制刷新；
- 另有 **GLM AI 解读**按钮（需配置 GLM API Key，解读内容仅供参考）。

#### ③ 实时行情
- 自选股实时报价面板（交易时段每秒级刷新）；
- 非交易时段显示最后收盘价与"行情连接中/已收盘"状态。

#### ④ 财务报表
- 输入股票代码查看三大报表历史数据（PIT 口径，按披露时点可见，避免未来函数）。

#### ⑤ 数据管理
- **实时行情同步守护进程**：启动 / 停止（交易时段建议开着）；
- **手动数据更新**："增量更新 K 线"（可限定股票范围，留空=全市场）、"刷新财务数据"；
- **更新进度**：进度条、起止时间、进程 PID；
- **数据覆盖**：股票总数、K 线总数、平均条数、最新交易日；
- **数据源健康 / 更新审计**：各数据源（TdxQuant/新浪/腾讯/AkShare/BaoStock）状态与历史更新记录。

> 所有"启动更新/停止/开关守护进程"都是控制面动作需令牌；浏览、榜单、K 线、报表查看均为只读免令牌。

---

### 6.10 市场资讯（金十 / 巨潮）

**入口**：数据与系统 → 市场资讯。2 个子标签：

#### 金十数据
4 类内容：**市场快讯**（实时滚动）、**新闻资讯**、**宏观行情**（全品种报价）、**财经日历**（高星事件、公布值/预期值）。
- 支持关键词搜索、"深度加载"更多页、强制刷新；
- 快讯右键/按钮可"复制原文"、调 **AI 解读**（interpret_flash，只读）；
- 快讯/新闻/行情/日历拉取属于控制面动作（未配令牌时可能 403），状态查看不受限。

#### 巨潮公告
- 官方信息披露查询，4 个分类：**最新公告 / 业绩快报 / 业绩预告 / 业绩公告**（年报/半年报/季报）；
- 可按 **6 位股票代码** + **标题/正文关键词**筛选，一键清除筛选；
- 每条结果可"打开巨潮官方原文"；
- 查询为只读，免令牌。

---

## 7. 日常使用流程（建议照做）

### 7.1 每个交易日收盘后（约 17:00 后）

```text
1. 跑每日数据更新（二选一）
   ① 双击 daily_update.bat
   ② 网页：市场数据 → 数据管理 → 增量更新 K线（需令牌）

2.（装了计划任务则全自动，否则手动）跑每日研究流水线：
   .venv-qlib\Scripts\python.exe scripts\run_daily_research_pipeline.py --workers 8
   它会自动完成：因子发布 → F4 就绪判定 → 研究选股（F4 不过则反弹为实验组合）→ 镜像同步

3. 17:10 起 F5 计划任务自动准备次日模拟计划（装了计划任务即可）

4. 次日开盘后打开"投资驾驶舱/模拟执行/模拟组合"查看自动模拟成交
```

### 7.2 每周例行（计划任务自动完成，手动命令备查）

| 时间 | 任务 | 手动入口 |
|---|---|---|
| 周六 18:30 | Qlib 周度窗口训练 | Qlib 实验页 → 模型训练（需令牌） |
| 周日 10:00 | F4 策略走样本外验证 | 命令行 `validate_strategy_portfolios.py`（经 scheduler） |
| 季度 | 财务数据全量刷新 | `daily_update.py --financial` |

### 7.3 休市日（周末 / 节假日）

- 页面显示 `market_session=closed`、`non_trading_weekday` 均为正常；
- 数据更新流水线内置交易日历门，休市日运行会直接判定无需更新；
- 适合做历史研究、跑 Qlib/F4 训练，但不会有模拟成交。

### 7.4 典型浏览路径（免令牌就能完成）

```text
登录(模拟盘操作员)
 → 驾驶舱：看权益、风险、数据新鲜度、今日研究组合
 → 市场数据/市场浏览：看 Top100、搜个股、看 K 线和逐笔
 → 因子研究/市场榜单：看因子排行榜和成分股
 → 策略运行：看 F4 状态和每日选股（注意：仅研究用途）
 → 模拟组合/模拟执行：看 100 万账本的持仓、订单、成交与对账
 → 风险监控：看组合风险与系统诊断
 → 市场资讯：看快讯、公告、财经日历
```

---

## 8. 命令行运维手册（进阶）

所有命令在项目根目录执行。普通数据脚本用系统 Python，研究/训练脚本建议用 `.venv-qlib`。

### 8.1 服务管理

```powershell
# 幂等启动 Web(8888)+API(8880)
node scripts\start_services.mjs

# 安装为 Windows 计划任务（开机自启 / 常驻看门）
powershell -ExecutionPolicy Bypass -File scripts\install_service_tasks.ps1
#   生成：XuanJiQuant-API-Service / -Web-Service / -Service-Supervisor
```

### 8.2 数据更新

```powershell
python scripts\daily_update.py                # 增量日K（约30分钟，收盘后）
python scripts\daily_update.py --financial    # 财务刷新（约5小时，季度）
python scripts\evaluate_factors.py            # 重评因子有效性（约30分钟）
python scripts\complete_ashare_data.py        # 全市场数据补齐/修复
```

### 8.3 每日研究流水线（一键编排）

```powershell
$env:PYTHON='d:\XuanJiQuant-v1.0.0-portable\.venv-qlib\Scripts\python.exe'
.\.venv-qlib\Scripts\python.exe scripts\run_daily_research_pipeline.py --workers 8
```

内部阶段（失败关闭，任一步不过则不推进）：

```text
research_training_scheduler (lane=daily)
  → factor_daily              因子发布（覆盖率门禁 ≥95%，身份哈希一致）
  → research_selection_daily  研究选股
       ├─ F4 candidate 合格 → generate_research_portfolio（正式研究组合）
       └─ F4 rejected 纯绩效失败 → generate_experimental_portfolio（10 只实验组合）
  → f4_readiness              current / stale / blocked 判定
  → mirror_sync               兼容镜像同步
  → shadow_research（可选）   存在 data/nautilus-baseline/baseline_result.json 时才跑影子周期
```

产物目录：`data\research\daily\generations\<generation_id>\`（selection.json、factor_snapshot 等）。

### 8.4 F4 策略验证

```powershell
# 通常由周日计划任务 / scheduler 调用；手动执行：
.\.venv-qlib\Scripts\python.exe scripts\validate_strategy_portfolios.py
```

- 读取四份输入：PIT manifest、质量报告、PIT 行业、沪深300基准 + 因子快照；
- 24 候选 × 5 个滚动窗口串行训练/验证/测试，全量约需数小时；
- 产物：`data\research\f4\latest.json`（轻量只读投影）与不可变证据目录；
- 相同输入重复运行会命中 no_op 不重训；输入身份变更则全新 run。

### 8.5 F5 模拟执行（计划任务安装）

```powershell
# 日频：交易日 17:10 准备次日计划（重试 3 次、最长 2 小时）
powershell -ExecutionPolicy Bypass -File scripts\install_f5_paper_task.ps1
# 盘中：09:35-11:25、13:05-14:50 每 5 分钟模拟成交
powershell -ExecutionPolicy Bypass -File scripts\install_f5_intraday_task.ps1
# 研究日程：每日 16:20 选股 + 周日 F4
powershell -ExecutionPolicy Bypass -File scripts\install_research_schedules.ps1
# Qlib 周度计划
powershell -ExecutionPolicy Bypass -File scripts\install_qlib_schedule.ps1
```

手动触发一次 F5 到期周期（需后端已配令牌，用文件传 JSON 体）：

```powershell
'{"action":"run_due"}' | Out-File $env:TEMP\f5.json -Encoding ascii -NoNewline
curl.exe -X POST http://127.0.0.1:8880/api/paper-execution `
  -H "Content-Type: application/json" `
  -H "X-XuanJi-Token: 你的令牌" `
  --data-binary "@$env:TEMP\f5.json"
```

只读查询不需要令牌：`{"action":"status"}`、`account`、`runs`、`orders`、`fills`、`positions`、`reconciliations`、`audit`。

### 8.6 六年 PIT 与 Qlib

```powershell
.\.venv-qlib\Scripts\python.exe scripts\collect_six_year_pit.py   # 采集六年数据
# 质量门禁 + 导出 bin 也可在网页 Qlib实验→数据准备 点按钮完成
```

- qlib bin 数据：`data\qlib\qlib_bin\a_share_6y_daily\`
- HDF5 数据集缓存：`data\qlib\qlib_cache\a_share_6y_daily\dataset\`（可删，删后自动重建、首跑变慢）

---

## 9. 核心概念通俗解释

| 概念 | 通俗解释 |
|---|---|
| **PIT（Point-in-Time，时点数据）** | 站在历史某一天只能看到"当时已经公布"的数据。比如年报 4 月才公告，就不许让 1 月的回测"提前看到"。系统用 PIT 数据 + as_of 截断严防这种未来函数 |
| **未来函数** | 用了当时不可能知道的未来信息，回测会虚高。F4 有自动检查，违规数必须为 0 |
| **走样本外（Walk-Forward）** | 用 504 天历史训练策略，在接下来的 126 天"比武选冠军"，再用之后 126 天**从没见过的行情**考核冠军。窗口不断向前滚，避免"在考卷上练习" |
| **purge 20 / embargo 5** | 训练和考核之间强制挖掉 20+5 个交易日的缓冲带，防止标签周期重叠造成的信息泄漏 |
| **F4 五大门槛** | ①正超额窗口比例 ≥60% ②成本后超额为正 ③夏普 ≥0.8 ④最大回撤不超过 −20% ⑤双倍成本下超额仍为正。任一不过即 rejected |
| **三档成本** | 测试按正常、1.5 倍、2 倍交易成本各跑一遍，防止策略只是"靠假设手续费低才赚钱" |
| **research_only** | 研究产物的固定标签：只供研究，永远不能直接变成交易指令 |
| **validated / experimental 车道** | F4 全过走 validated_paper 正式模拟；F4 因**纯绩效原因**被拒（但零违规、证据完整）可走 experimental_paper 实验模拟；数据有问题则 blocked 零订单 |
| **影子研究（Shadow）** | AI 在隔离环境里只读数据、产出"决策建议书"（DecisionProposal），四个权限布尔值全为 false，不碰账本、不触发下单 |
| **确定性** | 同样输入必然得到同样输出：固定时间触发、固定候选、固定参数、幂等重放，没有 AI 临时发挥 |
| **十项对账** | 每轮模拟成交后校验现金、持仓、权益、订单/成交一致性等 10 条不变量；任何一条对不上立即熔断 |
| **T+1 / 整手 / ADV** | 买入当日不可卖；A 股 100 股一手；单笔模拟成交量不超过该票当时市场成交量的 10% |
| **熔断（kill_switch）** | 系统发现无法自证账目正确时自动拉开的总开关，拉开后不再产生新订单，需人工排查 |

---

## 10. 数据文件与目录说明

```text
项目根目录\
├─ start_all.bat                一键启动（双击）
├─ daily_update.bat             每日数据更新（双击）
├─ .env                         环境变量（需从 .env.example 复制自建）
├─ components\                  前端页面源码
├─ server\                      Node 后端（端口 8880）
├─ quant\                       Python 量化核心
│   ├─ data\                    数据源/同步/质量门禁
│   ├─ factor\                  58 因子引擎、IC 评估
│   ├─ strategy\                F4 候选工厂、走样本外、选股
│   ├─ qlib\                    qlib 采集/训练/回测桥接
│   ├─ risk\                    硬风控
│   ├─ paper_execution\         F5 撮合、账本、滑点、对账
│   └─ valuation\               绝对/相对/市场估值
├─ scripts\                     各类 runner 与流水线脚本
├─ trading_system\              隔离基线与只读影子研究合同
├─ config\                      策略配置文件
└─ data\                        【运行产物，软件的"数据库"】
    ├─ quant.db                 SQLite 状态总线（行情缓存、任务、审计等，约数百 MB）
    ├─ factor_snapshot_latest.json   最新因子快照（页面因子榜单的数据源）
    ├─ factor_evaluation.json        因子有效性评估
    ├─ paper\f5_ledger.db       唯一活动模拟账本（100 万账户）
    ├─ qlib\
    │   ├─ qlib_bin\...         六年 qlib bin 数据
    │   └─ qlib_cache\...       HDF5 训练缓存（可重建）
    └─ research\
        ├─ f4\                  F4 验证证据（latest.json 为当前投影）
        ├─ daily\generations\   每日研究流水线产物（selection.json 等）
        ├─ industry\            PIT 行业分类
        └─ benchmarks\          沪深300 基准
logs\                           所有服务与任务日志（排查问题先看这里）
```

**重要：`data\` 目录是软件运行的全部积累，务必备份；删除等于重新初始化。**

---

## 11. 权限模型与 API 令牌配置

### 11.1 哪些操作免令牌（开箱即用）

所有**只读**操作不需要令牌：行情/K线/逐笔/财报浏览、因子榜单与列表、F4 与选股状态查看、Qlib 记录查看、风险/告警查看、资讯公告查询、F5 全部状态查询、估值分析等。

### 11.2 哪些操作需要令牌

所有**控制面（写入/运行）**动作：数据更新与守护进程开关、因子 IC 计算、策略/多因子回测运行、Qlib 数据导出与训练/取消/晋升、告警确认/解决/规则变更、金十快讯拉取、F5 手动 `run_due` 等。

未配置令牌时这些请求返回：

```text
HTTP 403 控制面请求未授权: 请使用本机、JSON Content-Type 并配置/发送 XUANJI_API_TOKEN
```

### 11.3 配置令牌的步骤（需要使用控制按钮时）

1. 复制模板：把 `.env.example` 复制为项目根目录的 `.env`；
2. 在 `.env` 中设置**两个相同的值**（后端校验与浏览器发送必须一致）：

   ```text
   XUANJI_API_TOKEN=你自己设定的一串足够长的随机字符
   VITE_XUANJI_API_TOKEN=与上面完全相同
   ```

3. 重启两个服务（关掉后重新 `node scripts\start_services.mjs` 或重新双击 start_all.bat）；
4. 刷新浏览器。之后页面控制请求会自动带上令牌。

### 11.4 其他可配置项（.env，按需）

| 变量 | 用途 |
|---|---|
| `GLM_API_KEY` / `GLM_BASE_URL` / `GLM_MODEL` | 估值/解读用的 GLM 大模型 |
| `OPENCODE_API_KEY` / `OPENCODE_MODEL` | 免费的 OpenCode Zen 模型 |
| `JIN10_MCP_BEARER_TOKEN` | 金十 MCP 鉴权（免费源可能需要） |
| `QUANT_CACHE` | 存储方式，默认 sqlite（可选 redis） |
| `PYTHON` | 强制指定后端调用的 Python 解释器路径 |
| `ALLOWED_ORIGINS` / `HOST` / `PORT` | 监听配置，默认仅本机 |

---

## 12. 常见问题 FAQ

**Q1：页面能打开，但很多按钮点了提示 403？**
正常。未配置 `XUANJI_API_TOKEN` 时所有写入/运行类按钮都会被拒绝，按第 11.3 节配置令牌并重启即可。纯浏览不受影响。

**Q2：驾驶舱显示"非交易日 / closed / non_trading_weekday"？**
今天是周末或节假日。系统内置交易日历，休市不更新、不成交是正确行为。

**Q3：因子榜单/策略页空白或一直加载？**
当天因子快照还没生成。收盘后先跑 `daily_update.py`，再跑 `run_daily_research_pipeline.py`（见第 7.1 节）。产物在 `data\research\daily\generations\`。

**Q4：F4 状态是 `f4_rejected`，是不是软件坏了？**
不是。这表示数据与流程全部合规（零未来函数、零违规），只是 24 个策略候选在交易成本后都没能跑赢基准，被门禁**诚实拒绝**。系统会自动改用实验组合继续模拟车道，这正是 F4 存在的意义——拦住"看起来很美"的策略。

**Q5：F4 状态是 `f4_blocked`？**
这才是异常：数据/身份门禁未通过（数据缺失、版本错配、行业或基准证据缺失）。按页面 reason_code 排查数据，重新跑数据更新与流水线。

**Q6：F5 没有订单/成交？**
依次检查：①是否交易日、是否在计划时间窗；②模拟执行页的"模拟执行许可/策略质量/执行通道/拒绝原因"；③F4 是否 blocked（blocked 必零订单）；④是否从未手动运行过 `run_due` 且未安装计划任务。只读信息都在模拟执行页。

**Q7：服务起不来 / 端口被占用？**
看 `logs\backend-launcher-err.log`、`logs\frontend-launcher-err.log`。8880/8888 被占时先查占用程序；`start_services.mjs` 是幂等的，服务已在运行时会直接提示 already listening。

**Q8：免费数据源拉取失败？**
免费源有限流和延迟。系统按 TdxQuant → 新浪/腾讯 → AkShare/BaoStock 的策略自动切换并记录健康度；偶发失败重试即可，覆盖率门禁不足 95% 时流水线会失败关闭，不会拿残缺数据硬算。

**Q9：能拿这个软件直接实盘炒股吗？**
不能，也不应尝试。系统不含券商接口，实盘权限在代码层面永久关闭；如需实盘必须另行建设合规券商通道并重新设计认证、风控与灾备。

**Q10：六年 Qlib 训练跑得很慢 / 中途电脑睡眠中断了怎么办？**
训练是纯 CPU 的重任务，建议接通电源并在电源选项里**禁止睡眠**（管理员 PowerShell：`powercfg /change standby-timeout-ac 0`）。已完成的窗口有防篡改锁，重跑会自动恢复校验、只补未完成窗口；HDF5 缓存建立后会越来越快。

---

## 13. 附录：API 端点 / 计划任务 / 配置文件速查

### 13.1 页面 ↔ API ↔ 功能对照

| 页面 | API 端点 | 只读 action（免令牌） | 控制面 action（需令牌，举例） |
|---|---|---|---|
| 驾驶舱 | `/api/workbench` | status | — |
| 市场数据 | `/api/data` | stocks, klines, financials, realtime_prices, ticks, indices, sector_flow, northbound, watchlist_get 等 | watchlist_add/remove/set/reset, tick_collect_full_day |
| 数据同步 | `/api/sync` | status, health, catalog, progress, daemon_status, stream_status | start_update, stop_update, 守护进程开关 |
| 因子研究 | `/api/factor` | meta, factors, market_eval, factor_stocks | evaluate, evaluate_segments, evaluate_all |
| 策略运行 | `/api/strategy` | meta, market_scan, research_selection | run（策略/多因子回测） |
| Qlib 实验 | `/api/qlib` | status, datasets, jobs, experiments, models, reports, backtests, workflow_runs, quality_reports 等 | setup, collect_six_years, quality_six_years, export_six_years, workflow_*, promote_shadow, cancel_job |
| 模拟执行 | `/api/paper-execution` | status, account, runs, orders, fills, positions, equity, reconciliations, audit | run_due, run_intraday |
| 历史模拟 | `/api/paper` | status, get_config, progress, log, report, benchmark | — |
| 执行投影 | `/api/execution` | all, status, positions, orders, trades | —（旧执行层仅只读） |
| 风险监控 | `/api/risk` | portfolio_risk, system_health, audit_replays, audit_replay, decision_trace, audit_logs | — |
| 告警中心 | `/api/alerts` | list, rules, stats | acknowledge, resolve, update_rule, silence（均须操作员+原因） |
| 市场资讯 | `/api/jin10` | status, tools, resources, codes | flash, news, quotes, calendar, interpret_flash |
| 巨潮公告 | `/api/cninfo` | query, detail, status | — |
| 股票估值 | `/api/valuation` | analyze, latest | glm_analyze |
| 实时指数 | GET `/api/market/indices`、SSE `/api/market-stream` | 免令牌 | — |

任意不在白名单内的下单动作统一返回 `409 arbitrary_order_action_forbidden`。

### 13.2 Windows 计划任务一览

| 计划任务 | 时间 | 职责 |
|---|---|---|
| XuanJiQuant-Research-Daily | 交易日 16:20 | 因子发布 + 研究选股快照 |
| XuanJiQuant-Qlib-Weekly | 周六 18:30 | Qlib 窗口隔离训练 |
| XuanJiQuant-Strategy-Weekly | 周日 10:00 | F4 走样本外验证 |
| XuanJiQuant-Paper-Daily | 交易日 17:10 | 准备下一交易日模拟计划 |
| F5 盘中任务 | 交易日 09:35–11:25 / 13:05–14:50 每 5 分钟 | 实时行情模拟成交 |
| XuanJiQuant-API/Web/Supervisor-Service | 开机/常驻 | 后端、前端托管与看门 |

所有任务使用 `IgnoreNew` 策略（上一轮没跑完不会重叠启动）。**安装任何计划任务前应先完成全量回归**：

```powershell
python -m pytest -q
npm run test:contracts
npx tsc --noEmit
npm run build
```

### 13.3 关键配置文件

| 文件 | 作用 |
|---|---|
| `.env`（由 .env.example 复制） | API 令牌、LLM Key、数据源令牌、端口、Python 路径 |
| `config/data_sync_policy.json` | 各数据集的主/备数据源、刷新频率、过期阈值（单一权威） |
| `config/f5_paper_execution.json` | F5 政策：初始资金、佣金/印花税/滑点、整手、实验车道开关 |
| `config/llm_models.json` | GLM / OpenCode 模型清单 |
| `quant/risk/hard_limits.json` | 组合硬风控限制（不可在线放宽） |

### 13.4 F5 模拟费用与撮合参数（默认值）

| 参数 | 值 |
|---|---|
| 初始资金 | 1,000,000 元 |
| 佣金率 / 最低佣金 | 万 3 / 5 元 |
| 印花税（卖出） | 千 0.5 |
| 过户费 | 十万分之一 |
| 滑点 | 万 1 |
| 成交量参与上限（ADV cap） | 10% |
| 每手股数 | 100 |
| 交收制度 | T+1 |

---

> **风险提示**：本软件仅用于量化研究、教学与模拟盘验证，不构成任何投资建议，不连接真实券商，不应直接用于实盘交易。市场有风险，回测与模拟结果不代表未来收益。
