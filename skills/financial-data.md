# 财务数据获取与交叉验证规范

本规范适用于所有涉及企业财务数据的研究。**每个关键数据必须来自两个独立来源，误差>1%须标记。**

---

## 📍 路径约定（读本文件前必看）

本文件里的命令使用三种路径，**各自相对什么，必须分清**：

| 写法 | 相对什么 | 例子 |
|------|---------|------|
| `~/.agents/skills/...` | **用户主目录**（`~`） | `cd ~/.agents/skills/tushare-data/scripts` |
| `tools/xxx.py` | **ai-berkshire 仓库根目录** | `python3 tools/twstock_data.py quote 2330` |
| `eastmoney.com` 等 | 网址，不是路径 | 直接访问 |

**⚠️ `tools/xxx.py` 不会从任意目录都可用。** 它相对**仓库根**，而 skill 被调用时
（从 `~/.agents/skills/financial-data/`）当前目录通常不是仓库根。
**执行 `tools/` 下的命令前，先定位仓库根：**

```bash
# 方法一：已经在仓库内（推荐）
cd "$(git rev-parse --show-toplevel)"

# 方法二：从任意位置定位（Windows）
cd /d D:\aiProject\ai-berkshire          # 本机实际路径

# 方法三：先确认，再执行
ls tools/twstock_data.py tools/ashare_data.py   # 都在才说明位置对了
```

**若报告里引用了 `tools/` 的输出，请写明执行时所在目录**，否则别人无法复现。

---

## 🔴 取数总链路（本机，先读这一节）

**本机装了多个取数工具。取任何数据前，按下面顺序尝试：前一环成功就不要走下一环。**

```
① tushare-data skill    A股行情/财务/行业/资金面/宏观（30个接口可用）
        ↓ 拿不到（无权限 / 通道不可用 / 不覆盖该数据）
② a-stock-data skill    A股全栈60端点，免token，不怕封IP（通达信/腾讯优先）
        ↓ 拿不到
③ tools/（本仓库）       台股 twstock_data.py；A股单只 ashare_data.py；验算 financial_rigor.py
        ↓ 拿不到
④ web_search 兜底        商品价格、政策解读、海外公司、行业供需 —— 只有这里能拿
```

### 第 ① 层：`tushare-data`（首选）

```bash
cd ~/.agents/skills/tushare-data/scripts      # ~ = 用户主目录
python tushare_client.py check        # 通道健康检查，必须先跑
python tushare_client.py call daily_basic --params '{"trade_date":"20260918"}'
python tushare_client.py call fina_indicator --params '{"ts_code":"601088.SH","start_date":"20240101","end_date":"20251231"}'
```

**凭据从 Windows 用户变量读取**（`TUSHARE_TOKEN` 在注册表 `HKCU\Environment`，进程环境通常为空；
`TUSHARE_URL` 指定自定义服务地址）。`tushare_client.py` 已实现读取顺序，
**不要自己写 token 逻辑，也不要绕过它直接用 `ts.pro_api(token)`** ——
tushare 官方包把地址写死为 `api.waditu.com`，本机 token 是签发给 `TUSHARE_URL` 的，
直连官方必然报「您的token不对」。

**✅ 第①层可用接口（30个）：** `daily` `daily_basic` `adj_factor` `income` `balancesheet`
`cashflow` `fina_indicator` `forecast` `stock_basic` `trade_cal` `stock_company`
`index_classify` `index_member_all` `sw_daily` `ths_index` `dc_index` `moneyflow`
`moneyflow_hsgt` `top_list` `margin_detail` `limit_list_d` `cn_cpi` `cn_ppi` `cn_pmi`
`sf_month` `cn_m` `fund_basic` `fund_daily` `index_global` `us_tycr`

**❌ 第①层无权限接口（8个）：** `npr`(政策库) `research_report`(研报) `anns_d`(公告)
`news` `major_news` `irm_qa_sh` `hk_daily`(港股) `us_daily`(美股)

> 这 8 个报「请联系管理员添加此权限」—— **是通道权限限制，不是接口不存在。**
> 不要重试，不要说「Tushare 没有这个数据」，直接降级到第②层或第④层。

### 第 ② 层：`a-stock-data`（A股全栈）

适用：第①层无权限的数据（公告/新闻/研报/ETF期权/打板/筹码分布）、
需要不怕封 IP 的稳定源（通达信 mootdx / 腾讯）、`is_stale` 僵尸报价、北交所官方行情。

```bash
# 见 ~/.agents/skills/a-stock-data/SKILL.md 的「端点路由速查」表
```

⚠️ **依赖现状：** `requests`/`pandas`/`numpy` 已装；
`mootdx`/`stockstats`/`baostock`/`xlrd`/`openpyxl` **未装**。
纯 HTTP 端点（腾讯/东财）无需这些包；用到达通信或 baostock 前先 `pip install mootdx baostock`。

⚠️ **东财接口有风控**（每秒>5次/单IP并发≥10 → 临时封IP）。批量调用必须走它的
`em_get()` 限流入口，或自带 ≥1s 间隔。

### 第 ③ 层：本仓库 `tools/`

**⚠️ 下表路径全部相对 ai-berkshire 仓库根目录。执行前先 `cd` 到仓库根**（见文首「路径约定」）。

| 工具 | 用途 | 依赖 |
|------|------|------|
| `tools/twstock_data.py` | **台股** FinMind 取数（6个命令，自带市值验算） | 零依赖 |
| `tools/ashare_data.py` | A股**单只**行情/财务/估值/搜索 | 零依赖 |
| `tools/financial_rigor.py` | 精确计算与验算（市值/估值/三情景） | 零依赖 |
| `tools/report_audit.py` | 报告数据抽检 | 零依赖 |

**从仓库根执行（正确）：**

```bash
cd "$(git rev-parse --show-toplevel)"        # 定位仓库根
python3 tools/ashare_data.py quote 601088
python3 tools/ashare_data.py financials 601088
```

**⚠️ 常见错误：** 从 `~/.agents/skills/financial-data/` 直接跑 `python3 tools/ashare_data.py`，
会报 `No such file or directory` —— 因为该目录下没有 `tools/`。
**若无法确定仓库根，就用绝对路径调用**（仅本机可用，不可写进报告）：
`python3 D:/aiProject/ai-berkshire/tools/ashare_data.py quote 601088`

### 第 ④ 层：`web_search` 兜底

**以下数据前三层都拿不到，只能走 web_search：**
- **商品价格**（煤炭、钢材、化工品、农产品等大宗商品价格）
- **行业供需**（产量、库存、进口、开工率）
- **政策原文解读**（`npr` 无权限）
- **海外公司**（`hk_daily`/`us_daily` 无权限；美股的
  macrotrends/stockanalysis 仍按下方美股章节走）
- **未上市公司**

### ⚠️ 来源必须如实标注

**报告中必须写清每个数字的实际来源**（「来源：Tushare `daily_basic`」/
「来源：东方财富 `clist`」/「来源：web_search」）。
**不得把降级后的来源伪装成 Tushare，也不得把搜索来的数字说成接口数据。**

---

## 数据源优先级（按市场）

### 美股（PDD、腾讯ADR、网易ADR等）

| 优先级 | 来源 | URL | 获取方式 |
|--------|------|-----|---------|
| 1（主） | **macrotrends** | macrotrends.net/stocks/charts/{ticker} | 直接访问，无需注册 |
| 2（副） | **stockanalysis** | stockanalysis.com/stocks/{ticker}/financials | 直接访问，无需注册 |
| 原始一手 | SEC EDGAR | sec.gov/cgi-bin/browse-edgar | 10-K / 10-Q 原文 |

### 港股（腾讯0700、网易9999、美团3690等）

| 优先级 | 来源 | URL | 获取方式 |
|--------|------|-----|---------|
| 1（主） | **aastocks** | aastocks.com/tc/stocks/analysis/company-fundamental | 直接访问 |
| 2（副） | **macrotrends**（ADR代码） | 腾讯用TCEHY，网易用NTES | 直接访问 |
| 原始一手 | HKEX披露易 | hkexnews.hk | 年报PDF |

### A股（三七互娱、吉比特等）

**⚠️ 先走上方「取数总链路」的第①层（tushare-data）。下表是链路走不通时的信源对照。**

| 优先级 | 来源 | URL | 获取方式 |
|--------|------|-----|---------|
| **1（主）** | **Tushare** | `tushare_client.py`（见总链路） | 接口调用，30个可用 |
| **2（副）** | **东方财富** | eastmoney.com → 搜股票代码 → 财务报表 | 直接访问 / `tools/ashare_data.py financials` |
| 原始一手 | **巨潮资讯** | cninfo.com.cn | 原始年报/季报PDF |
| **批量/行业** | **东方财富行业板块** | `push2delay.eastmoney.com/api/qt/clist/get` | 见下 |
| **政策/研报/公告** | ⚠️ Tushare 无权限 → 走 `a-stock-data` 或 web_search | | |

**A股常用接口速查（第①层可用）：**

| 要什么 | 接口 | 关键参数 |
|--------|------|---------|
| 日线行情 | `daily` | `trade_date=YYYYMMDD` 或 `ts_code`+`start_date`+`end_date` |
| 估值(PE/PB/市值) | `daily_basic` | `trade_date` 或 `ts_code` |
| 利润表 | `income` | `ts_code`+`start_date`+`end_date` |
| 资产负债表 | `balancesheet` | 同上 |
| 现金流量表 | `cashflow` | 同上 |
| 财务指标(ROE/毛利率) | `fina_indicator` | 同上 |
| 复权因子 | `adj_factor` | 跨除权日比价必用 |
| 股票列表 | `stock_basic` | `list_status='L'` |
| 申万行业分类 | `index_classify` | `level='L1'`, `src='SW2021'` |
| 申万行业成分 | `index_member_all` | `l1_name='煤炭'` |
| 东财板块 | `dc_index` | `trade_date` |
| 个股资金流 | `moneyflow` | `trade_date` |
| 龙虎榜 | `top_list` | `trade_date` |

**⚠️ A股批量取数（行业研究用）：**
Tushare 的 `stock_basic` 一次能给全市场 5568 只股票及其 `industry` 字段，
**比手工列名单可靠** —— 手工名单无法自证完整。

**东方财富行业板块接口（第②层备选，免 token）：**
```
push2delay.eastmoney.com/api/qt/clist/get
  ?fs=b:BK0437        # 板块代码，BK0437=煤炭
  &fields=f12,f14,f20,f3,f9,f23
  &pz=300&pn=1&po=1&np=1&fltt=2&invt=2&fid=f20
```
⚠️ **`push2.eastmoney.com` 会限流，必须回退 `push2delay.eastmoney.com`，并每次间隔 ≥1s。**

### 台股（台积电2330、联发科2454、大立光3008等）

| 优先级 | 来源 | URL | 获取方式 |
|--------|------|-----|---------|
| 1（主） | **FinMind API** | api.finmindtrade.com | `tools/twstock_data.py`（零依赖脚本，见下） |
| 2（副） | **Goodinfo台湾股市资讯网** | goodinfo.tw/tw/StockDetail.asp?STOCK_ID={代码} | 直接访问 |
| 原始一手 | 公开资讯观测站（MOPS） | mops.twse.com.tw | 财报原文/月营收公告 |

**FinMind 取数工具**（分析台股时优先调用，输出自带市值验算）：

> ⚠️ **下面命令相对 ai-berkshire 仓库根执行**，先 `cd "$(git rev-parse --show-toplevel)"`。
> 从别处直接跑会报 `No such file or directory`，原因见文首「路径约定」。

```bash
python3 tools/twstock_data.py quote 2330        # 最新行情 + PER/PBR/殖利率 + 市值验算
python3 tools/twstock_data.py valuation 2330    # 估值指标 + PER一年区间 + 52周高低
python3 tools/twstock_data.py financials 2330   # 近5年年度核心财务（营收/毛利率/归母净利/EPS/ROE）
python3 tools/twstock_data.py revenue 2330      # 近13个月月营收及同比
python3 tools/twstock_data.py dividend 2330     # 近年股利政策（现金/股票股利、除息日）
python3 tools/twstock_data.py search 台積        # 搜索股票代码（注意台股名称为繁体）
```

台股特别注意：

1. **货币单位是新台币（TWD）**，与港币/人民币/美元混排时必须显式标注，跨市场对比先统一换算
2. **月营收是台股独有优势**：上市柜公司每月10日前强制披露上月营收，是跟踪基本面拐点最快的公开信号，earnings-review/thesis-tracker 类分析应优先利用（`revenue` 子命令）
3. FinMind 损益表为**单季值**，工具已自动加总为年度值；不足4季的年份会标注"仅前N季累计"
4. FinMind 未注册可直接用（有小时级限额）。注册后的 API token **只存本机、严禁提交到 git**，工具按优先级自动读取：①环境变量 `FINMIND_TOKEN`；②本地文件 `local/finmind_token.txt`（`local/` 已被 `.gitignore` 永久排除，把 token 单独一行写入该文件即可）。token 不得出现在报告、skill、commit 中
5. 交叉验证：FinMind 数值与 Goodinfo（或 macrotrends 上的 ADR，如 TSM）对照，误差规则同下；台积电等有 ADR 的公司注意 ADR 与台股原股的汇率/存托比率差异（1 TSM ADR = 5 股 2330）

---

## 执行规范

### 第一步：获取数据

对每个财务指标（收入、净利润、毛利率、经营现金流、资产负债率等），分别从**来源1**和**来源2**取数。

### 第二步：误差计算与标记

```
误差率 = |来源1数值 - 来源2数值| / 来源1数值 × 100%
```

| 误差 | 处理方式 |
|------|---------|
| ≤ 1% | ✅ 一致，取来源1数值，标注两个来源 |
| 1% ~ 5% | ⚠️ 标记"数据存在差异"，注明两个数值，说明可能原因（汇率/会计口径） |
| > 5% | ❌ 标记"数据存在重大差异"，必须查原始财报核实，不得直接使用 |

### 第三步：数据呈现格式

每个关键数据必须按以下格式标注：

```
收入：1,239亿元 ✅
  - macrotrends: 1,241亿元
  - stockanalysis: 1,237亿元
  - 误差: 0.3%
```

差异示例：
```
净利润：245亿元 ⚠️ 数据存在差异
  - macrotrends: 245亿元（GAAP）
  - stockanalysis: 278亿元（Non-GAAP）
  - 误差: 13.5% — 原因：会计口径不同（GAAP vs Non-GAAP）
```

---

## 常见差异原因（不一定是数据错误）

| 原因 | 说明 |
|------|------|
| GAAP vs Non-GAAP | 最常见，尤其是利润类数据 |
| 汇率换算 | 港币/人民币/美元换算时间点不同 |
| 财年定义 | 自然年 vs 财年（如苹果财年10月结束） |
| 合并口径 | 是否含少数股东权益 |
| 数据更新滞后 | 某平台尚未更新最新一期财报 |

---

## 特别规则

1. **未上市公司**（米哈游、莉莉丝等）：只有一手数据来源时，数据前标记 `[估计]`，不执行交叉验证
2. **季度数据 vs 年度数据**：优先使用年度数据做交叉验证，季度数据部分来源可能有滞后
3. **原始财报优先**：若两个来源均与原始财报（10-K/年报PDF）不符，以原始财报为准，标记来源错误

---

## 股价与复权（历史序列必读）

价格有三种口径，混用会让历史股价位置、长期涨幅、历史估值分位全部失真：

| 口径 | 含义 | 用途 |
|------|------|------|
| 不复权 | 实际成交价，除权除息日跳空 | 仅用于"当前时点"快照 |
| 前复权 | 以最新价为基准回调历史价 | 历史股价对比、N年涨幅、历史PE band 一律用它 |
| 后复权 | 以上市首日为基准前推 | 计算历史总回报/年化收益 |

规则：

1. 涉及历史价格的分析统一用**前复权**，且同一分析内**不得混用**复权与不复权来源。
2. 当前市值/当前PE 用**当前实际股价 × 当前总股本**即可，与复权无关——复权只影响历史序列。
3. 跨越拆股/大比例送转的每股指标（历史EPS、历史股价），必须复权还原后再同比。
4. 总回报/年化收益需计入分红（后复权已含），只看价格涨幅会低估。
5. 增发/回购后市值验算以最新总股本为准（`financial_rigor.py verify-market-cap` 偏差>5% 会提示核对）。

---

## 快速索引

| 场景 | 主要来源 | 备用来源 |
|------|---------|---------|
| PDD / 拼多多 | macrotrends.net/stocks/charts/PDD | stockanalysis.com/stocks/pdd |
| 腾讯 | macrotrends.net/stocks/charts/TCEHY | aastocks（0700.HK） |
| 网易 | macrotrends.net/stocks/charts/NTES | aastocks（9999.HK） |
| 三七互娱 | eastmoney.com（002555） | cninfo.com.cn |
| 吉比特 | eastmoney.com（603444） | cninfo.com.cn |
| Nintendo | macrotrends.net/stocks/charts/NTDOY | stockanalysis.com/stocks/ntdoy |
| Capcom | macrotrends（CCOEY） | stockanalysis（CCOEY） |
| 台积电 | tools/twstock_data.py（2330） | goodinfo.tw / macrotrends（TSM，注意1 ADR=5股） |
| 联发科 | tools/twstock_data.py（2454） | goodinfo.tw |
