# Tangdouren 优惠券生成与核销系统 · 交付说明

对应 PRD：《Tangdouren 优惠券生成与核销系统 PRD》MVP v1.0，另含开发期确认的一处 v1.1 变更（活动前缀，见第 5.1 节）
仓库：`KeMiaoDaDi/Tangdouren`
基准提交：`fa00456`（交付复核时仍是默认分支 main 的顶端）
开发分支：`feat/coupon-system`
本地路径：`~/Desktop/Projects/Tangdouren`

交付分支相对基线共 4 笔提交：

| 提交 | 内容 |
| --- | --- |
| `b77b58f` | 优惠券系统本体（11 个文件，迁移 013） |
| `c3dea24` | 安全修复：为 `timer_sessions`、`blocked_time_slots` 启用 RLS（迁移 014） |
| `acda6e4` | 活动前缀（6 个文件，迁移 015） |
| `1778db4` | 后台侧边栏把「优惠券」入口移到列表末尾 |

相对基线合计：13 个文件，+2180 / −59。

---

## 1. 改动清单

### 提交一 `b77b58f`：优惠券系统本体

### 新增

| 文件 | 作用 |
| --- | --- |
| `supabase/migrations/013_coupons.sql` | coupons 表、约束、索引、RLS、timer_sessions 快照字段、结算核销与撤销结算两个事务函数 |
| `lib/coupon/coupon.ts` | 纯函数：类型/状态判定、优惠码规范化、便士换算、伦敦时区截止日换算、生成参数校验、**唯一的优惠计算入口 `computeSettlement`** |
| `lib/coupon/code.ts` | 服务端随机优惠码生成（`node:crypto`，排除 0/O、1/I；提交一为 `TD` + 8 位随机，提交三起改为「前缀 + 5 位随机」） |
| `lib/coupon/service.ts` | 读取订单+优惠券上下文、预览、RPC 错误 → 中文文案映射 |
| `app/api/admin/coupons/route.ts` | `GET` 列表（分页/搜索/状态筛选/统计）、`POST` 批量生成 |
| `app/api/admin/coupons/validate/route.ts` | `POST` 优惠码预验证（只预览，不核销） |
| `app/(admin)/dashboard/coupons/page.tsx` | 优惠券管理页（统计、搜索、筛选、列表、复制、CSV、生成弹窗） |
| `tests/coupon.test.ts` | 39 项纯函数测试（PRD 11.1 / 11.2 的可离线部分；提交三起含前缀用例） |

### 修改

| 文件 | 改了什么 |
| --- | --- |
| `app/api/admin/timers/[id]/route.ts` | `settle` 改为服务端计算金额并调用 RPC 原子核销（忽略客户端传入金额）；新增 `unsettle`；`DELETE` 对已核销订单返回中文提示 |
| `app/(admin)/dashboard/timers/[id]/page.tsx` | `SettlementPanel` 改造：去掉实收金额输入框，加优惠码验证/预览/最终应收/撤销结算（二次确认）；已结算展示原价、券码、优惠内容、优惠金额、实收、结算时间、结算人 |
| `components/admin/Sidebar.tsx` | 新增「优惠券」菜单入口 `/dashboard/coupons` |

改动规模：3 个已跟踪文件 +205 / −59，另新增 8 个文件。

### 提交二 `c3dea24`：RLS 安全修复

只含一个新文件 `supabase/migrations/014_enable_rls.sql`（两行 `ALTER TABLE ... ENABLE ROW LEVEL SECURITY`）。单独一笔提交，便于维护方单独取舍；背景与实测见第 7 节。

### 提交三 `acda6e4`：活动前缀（PRD v1.1 变更）

新增

| 文件 | 作用 |
| --- | --- |
| `supabase/migrations/015_coupon_prefix.sql` | `coupons.code_prefix` 列、格式与一致性 CHECK、索引，以及前缀用量统计函数 `coupon_prefix_usage()` |

修改

| 文件 | 改了什么 |
| --- | --- |
| `lib/coupon/code.ts` | 随机后缀由 8 位下调为 5 位；生成函数接受前缀（`generateCouponCode(prefix)` / `generateCouponCodes(quantity, prefix)`） |
| `lib/coupon/coupon.ts` | 新增 `normalizeCodePrefix` / `isValidCodePrefix` / `codePrefixHints`；生成参数校验新增 `codePrefix`（留空回退 `TD`，填错则明确报错而非静默吞掉） |
| `app/api/admin/coupons/route.ts` | `POST` 接收 `codePrefix` 并落库；`GET` 新增 `prefix` 筛选与 `usedPrefixes`（后者走 `coupon_prefix_usage()`，因为本项目 PostgREST 禁用了聚合函数） |
| `app/(admin)/dashboard/coupons/page.tsx` | 生成弹窗加前缀输入框、历史前缀快捷标签、非阻断提示；列表加前缀筛选与展示、每行「再生成一批」；导出 CSV 增加 `code_prefix` 列 |
| `tests/coupon.test.ts` | 新增 10 项前缀用例（格式、合法性、提示、回退 `TD`、长度上限） |

改动规模：5 个已跟踪文件 +372 / −21（含测试），另新增 1 个文件。

### 提交四 `1778db4`：侧边栏入口位置

`components/admin/Sidebar.tsx`：优惠券入口由列表中部移到末尾。基线本来只有五项，新增的入口改为追加在末尾，避免在中间插行。

---

## 2. 数据库迁移怎么执行

本次有三个迁移文件，都是**纯增量、向前兼容**的变更（只用 `CREATE TABLE IF NOT EXISTS` / `ADD COLUMN IF NOT EXISTS` / `CREATE INDEX IF NOT EXISTS` / `DROP CONSTRAINT IF EXISTS` 后重建），不修改任何旧迁移，可在现有 Supabase 项目直接执行。

**执行顺序：`013` → `014` → `015`，其中 `015` 必须在 `013` 之后。** `015` 依赖 `coupons` 表，CHECK 还会校验 `code_prefix` 与 `code` 开头一致；**不跑 `015` 就生成优惠券，会直接报缺列。** `014` 是两行 RLS 开关，与另外两个相互独立，只要求 `timer_sessions`、`blocked_time_slots` 已存在（迁移 008 / 011 已跑）。

三种方式任选：

**A. Supabase 控制台（最省事）**
Dashboard → SQL Editor → 粘贴 `013_coupons.sql` 全文 → Run。执行前建议先建一个数据库快照（Dashboard → Database → Backups）。

**B. Supabase CLI**
```bash
supabase link --project-ref <项目 ref>
supabase db push        # 会按顺序执行未应用的迁移，包括 013
```

**C. 直连 psql**
```bash
psql "postgresql://postgres.<ref>:<你的数据库密码>@<host>:5432/postgres" -f supabase/migrations/013_coupons.sql
```

三个迁移都用同样方式执行，按 `013` → `014` → `015` 的顺序；用 Supabase CLI 的话 `supabase db push` 会按文件名顺序自动依次执行。

执行后建议自检一次：
```sql
select count(*) from coupons;                          -- 应为 0
select column_name, data_type from information_schema.columns
 where table_name = 'timer_sessions'
   and column_name like any (array['%coupon%','%discount%','pre_discount%']);
select column_name from information_schema.columns
 where table_name = 'coupons' and column_name = 'code_prefix';   -- 015 跑完应有 1 行
```

迁移做的事：
1. 建 `coupons` 表（`code` 大写唯一、核销字段成组约束、`time_minutes` 必须是 30 的整数倍、`percentage_off < 100`）。
2. 对非空 `redeemed_session_id` 建唯一索引 → 一笔订单最多绑定一张券。
3. `redeemed_session_id` 外键 `ON DELETE RESTRICT` → 已核销订单不能被直接删除。
4. 启 RLS，仅 authenticated 可访问（实际 API 用 service role）。
5. 给 `timer_sessions` 加优惠快照字段。
6. 建两个事务函数 `settle_timer_session` / `unsettle_timer_session`，`SECURITY DEFINER` 且已 `REVOKE FROM PUBLIC`，只授权给 `service_role`（前端无法通过 PostgREST 直接调用）。
7. `014_enable_rls.sql`：对 `timer_sessions`、`blocked_time_slots` 各一行 `ENABLE ROW LEVEL SECURITY`，**不建任何策略**（即 anon / authenticated 完全无权访问），只堵住公开 key 的读、改、删。
8. `015_coupon_prefix.sql`：给 `coupons` 加 `code_prefix TEXT NOT NULL DEFAULT 'TD'`；CHECK 限制 `^[A-Z0-9]{2,8}$`，并额外校验 `left(code, length(code_prefix)) = code_prefix` 以防漏填（存量券由默认值覆盖回填，无需 UPDATE）；建 `code_prefix` 索引；建统计函数 `coupon_prefix_usage()`（`SECURITY DEFINER`，只授权 `service_role`），供后台统计与「历史前缀」快捷选择。

---

## 3. 测试结果

| 项目 | 命令 | 结果 |
| --- | --- | --- |
| 纯函数测试 | `node --test tests/*.test.ts` | 44 passed / 0 failed（其中优惠券 39 项，原有 5 项未受影响） |
| 类型检查 | `npx tsc --noEmit` | 通过，0 error |
| 生产构建 | `npm run build` | 通过，`/dashboard/coupons`、`/api/admin/coupons`、`/api/admin/coupons/validate` 均已产出 |
| ESLint | `npm run lint` | **基线就不可用**，见下 |
| 真实数据库端到端 | `coupon_verify.py`（对 Supabase 测试项目） | **80 passed / 0 failed**，见 3.1 |
| 匿名攻击复测 | `Tangdouren-匿名攻击复测脚本.py`（同一测试项目） | **15 passed / 0 failed**，见 3.1 |

覆盖到的 PRD 用例：
* 计算：£13.99 用 £2 券 → £11.99；券额大于原价 → £0.00；£25.99 用 15% OFF → 优惠 £3.90、应收 £22.09；60 分钟订单拒绝时长券；时长券统一按半小时续时价抵扣。
* 状态：永久券可用；伦敦时间截止日 23:59 前可用、之后过期；冬季 GMT 换算；过期券/已用券被拒；优惠码大小写与空格规范化。
* 参数：时长券非 30 倍数被拒；百分比越界被拒；固定金额小数位与数量上限校验；截止日期不得早于今天。
* 优惠码：格式、字符集（不含 0/O/1/I）、批量 500 张不重复。
* 前缀：2 到 8 位大写字母数字、自动转大写去空格、非法字符与非字符串输入被拒、留空回退 `TD`、生成后不可改、前缀含 `0/O/1/I` 时给非阻断提示。
* 后缀：长度 5 位、字符集仍不含 `0/O/1/I`、同前缀存量券的撞码由唯一索引 + 重试兜底。

**注册测试命令的提醒**：仓库 `package.json` 里没有 test script，既有测试是用 `node --test tests/*.test.ts` 跑的（`node --test tests/` 会因为把目录当测试文件而失败）。如果希望正式化，可以加一条 `"test": "node --test tests/*.test.ts"`（本次未擅自添加）。

**ESLint 说明**：上游仓库没有 ESLint 配置文件（`npm run lint` 在 `fa00456` 上就直接报 “ESLint couldn't find a configuration file”），与本改动无关。我用临时配置 `{"extends": ["next/core-web-vitals"]}` 对全仓库跑了一遍 eslint，本改动涉及的所有文件**零告警**；全库只有一处既有报错，在 `app/(admin)/login/page.tsx:103`（`<a>` 应改用 next/link 的 `<Link>`），该文件本次未改动。若要满足「lint 通过」的验收项，需要在仓库里补一份 `.eslintrc.json`（内容即上面两行），我没有擅自新增这个文件。

---

### 3.1 真实数据库端到端验证（Supabase 测试项目 `rlakqwmedlgpcdasvads`）

`013_coupons.sql`（连同 001–012）已在一个全新的 Supabase 测试项目上完整执行，之后 014、015 也在同一项目执行过。随后用 `coupon_verify.py` 绕过前端、直接通过 REST + RPC 压测数据库层，**80 项断言全部通过**：

| 组 | 覆盖内容 | 结果 |
| --- | --- | --- |
| 0 迁移就位 | `coupons` 表、两个事务函数、`timer_sessions` 四个快照字段、`coupons.code_prefix` 列、`coupon_prefix_usage()` | 9/9 |
| 1 数据库约束 | 占比 100、时长非 30 倍数、优惠值 0、小写券码、重复券码，全部被数据库拒绝 | 9/9 |
| 1b 活动前缀约束 | 前缀 1 位、9 位、非法字符、小写、与 `code` 开头不一致、中文，六项被数据库拒绝；2 位与 8 位边界可插入 | 8/8 |
| 2 无券结算 | 实收 = 原价、优惠 0、`pre_discount` 落库、重复结算被拒 | 6/6 |
| 3 固定金额券 | £13.99 − £2 = £11.99、快照落库、券被核销、二次使用被拒 | 6/6 |
| 4 撤销结算 | 结算字段清空、券恢复未使用、恢复后可用于另一订单、未结算不能撤销 | 9/9 |
| 5 百分比券 | £25.99 用 15% → 优惠 £3.90 / 应收 £22.09；金额对不上时数据库拒绝 | 4/4 |
| 6 时长券 | 60 分钟订单拒绝；90 分钟订单减 £6.00 → 应收 £13.98 | 3/3 |
| 7 券与订单状态 | 过期券、不存在的券、未结束计时，均被拒 | 3/3 |
| 8 并发 | 同一张券被两单并发核销：恰好一单成功；同一订单并发结算：恰好一次成功 | 4/4 |
| 9 外键保护 | 已核销订单不能删除；先撤销后可以删除 | 2/2 |
| 10 匿名访问封堵 | `timer_sessions` 匿名读 0 条、`coupons` 匿名读不到、`coupon_prefix_usage` 匿名调不到 | 3/3 |
| 11 活动前缀功能 | 前缀用量聚合与最近使用时间、按前缀筛选、留空回退 `TD`、带前缀券走完核销、同前缀撞码被唯一索引拦下 | 12/12 |
| 清理 | 测试数据全部删除，无残留 | 2/2 |

并发部分是真的两个线程同时发起 RPC（`threading.Barrier` 对齐起跑），不是串行模拟；数据库侧 `SELECT ... FOR UPDATE` + 部分唯一索引把抢券挡住了。

验证脚本保留在 `~/Desktop/Projects/Tangdouren-工具/优惠券系统-验证脚本.py`，改完代码可以重跑：

```bash
python3 ~/Desktop/Projects/Tangdouren-工具/优惠券系统-验证脚本.py https://<ref>.supabase.co <secret_key>
```

**验证中发现的一个操作要点**：`timer_sessions.coupon_id → coupons` 和 `coupons.redeemed_session_id → timer_sessions` 这两条外键都是 `ON DELETE RESTRICT`，**互相锁死**。也就是说券一旦被核销，订单和券**谁都删不掉**，必须先撤销结算（`unsettle` 在同一事务里把两侧引用一起清空）才能删除。这是 PRD 设计的直接结果、不是缺陷，但店员和运维需要知道这个操作顺序。

---

## 4. 环境变量

**不需要新增任何环境变量。** 继续使用原有的：

```
NEXT_PUBLIC_SUPABASE_URL
NEXT_PUBLIC_SUPABASE_ANON_KEY
SUPABASE_SERVICE_ROLE_KEY
```

service role key 只在服务端 API 使用，未进入任何客户端组件。

---

## 5. 与 PRD 的差异（已确认的产品决定）

| 项 | PRD 原文 | 实际实现 |
| --- | --- | --- |
| 时长券算法 | 减免后重新跑套餐计价（`calcBill(原分钟数 − 减免分钟数)`） | **改为统一按半小时续时价抵扣**：30 分钟券 = £6.00，60 分钟券 = £11.99，与订单时长无关。首小时仍不可减免（计费 ≤ 60 分钟的订单直接拒绝）。这样避免出现「150 分钟订单用 30 分钟券只省 £0.01」这类怪结果 |
| 人民币实收 | 未提及 | 保留 `actual_amount_cny` 字段，界面已无录入入口；已结算订单若历史上有值仍会展示 |
| 撤销结算 | 未说明关联预约如何处理 | 撤销结算**不回退**预约状态，预约仍保持「已完成」，需要人工在预约管理页处理 |
| `discounted_billing_minutes` | PRD 7.2 要求新增该字段 | 字段已建，但时长券不再重算计费时长，写入恒为 `NULL`（保留列备用） |
| 优惠码自定义 | PRD v1.0 §2 定「优惠码系统自动随机生成，不允许店员自定义」，§10 把「自定义公开优惠码」「活动、渠道、批次或来源管理」列在范围外 | **v1.1 变更**：前缀（2 到 8 位大写字母数字）由店员按活动自定义，如新生周 `WELCOME`、中秋 `LUCKY`；后缀仍由系统随机生成。不做顺序号（递增数字可被枚举）、不做整串自定义、不做生成后改前缀 |
| 随机后缀长度 | PRD 未规定长度，v1.0 实现为 8 位 | 下调为 5 位。理由见下方 5.1 |

一处需要留意的小边界：时长券改成定额抵扣后，90 分钟订单用 30 分钟券的结果是 £13.98，比「1 小时价」£13.99 低 1 便士。金额上无实际影响，但如果希望硬性保证「减免后不低于 1 小时价」，需要再加一条下限规则（当前未加）。

### 5.1 活动前缀规则（PRD v1.1 变更）

| 项 | 规则 |
| --- | --- |
| 前缀长度 | 2 到 8 位 |
| 允许字符 | 大写 `A-Z`、`0-9`；自动转大写、去首尾空格；其他字符拒绝 |
| 默认值 | `TD`（不填 = 旧行为，向后兼容） |
| 随机后缀 | 5 位，字符集剔除 `0/O/1/I`（32 个字符） |
| 非阻断提示 | 前缀含 `0/O/1/I` 时提示「顾客口头报号易串」 |
| 生成后 | 不可改前缀 |
| 唯一性 | `code` 唯一索引 + 插入冲突重试兜底 |

数据侧：`coupons.code_prefix`（`NOT NULL DEFAULT 'TD'`）+ CHECK（`^[A-Z0-9]{2,8}$`，并校验 `left(code, length(code_prefix)) = code_prefix`）+ 索引；存量券由默认值覆盖回填，无需 UPDATE。后台生成弹窗会列出已用过的前缀（数据来自 `coupon_prefix_usage()`），点一下即填入；列表页可按前缀筛选，导出 CSV 增加 `code_prefix` 列。

**后缀为什么取 5 位（有人问就照这个答）**：本系统没有面向顾客的核销入口，券码只在后台由店员输入验证，顾客无法自行提交，所以枚举风险只限于「收银台瞎报号」。真正的约束是撞上库里同前缀旧券的概率，实测同前缀存量 5000 张时 5 位整批失败率 0.037%，4 位则达 74.8%，所以取 5。

---

## 6. 待办与风险

1. **生产库执行迁移前先备份**（Dashboard → Database → Backups）。013 已在测试项目验证通过（见 3.1），但生产库有真实数据，且执行本身不可回滚。
2. **HTTP 接口层与前端页面的人工走查还没做**：优惠码验证预览、复制/CSV、结算面板的撤销二次确认。数据库层已全覆盖，这一层需要登录后台点一遍（在测试项目上走查要先建一个管理员账号）。PRD 里"预验证不会消耗优惠券"这一条由结构保证：`/api/admin/coupons/validate` 只做 SELECT + 纯函数计算，没有任何写路径。
3. **匿名可达面已实测封住**：用测试项目的 publishable key（`sb_publishable_...`）跑了 15 项匿名读 / 改 / 删复测，全部被挡，明细见第 7 节。生产库是否与迁移定义一致（可能被手工开过 RLS）仍待维护方确认，见第 7 节末。
4. **回归测试（PRD 11.3）需要在真机上跑**：开始/暂停/继续/结束计时、无券结算、预约自动标记完成、顾客端计时页与价格展示。
5. **订单/券的删除顺序**：已核销券的订单与券互相 RESTRICT 锁死（详见 3.1 末尾），必须先撤销结算才能删除；`DELETE` 接口已返回中文提示。
6. ESLint 配置缺失，`npm run lint` 在基线上就不可用（见第 3 节）。
7. 本机 3000 端口被 Hermes 自带的 WhatsApp bridge 常驻占用（只监听 127.0.0.1），`npm run dev` 会静默改用 IPv6 共用该端口，浏览器访问 `localhost:3000` 可能命中 bridge 而不是 Next。本地手工测试时建议 `npm run dev -- -p 3002`。

---

## 7. 安全发现：`timer_sessions` 与 `blocked_time_slots` 未启用 RLS（独立于本次需求，建议单独修复）

**这是既有代码的问题，不是本次优惠券改动引入的**，但在验证过程中被实测确认，严重程度高，建议尽快处理。

**根因**：迁移 008（`timer_sessions`）和 011（`blocked_time_slots`）建表时都没有 `ENABLE ROW LEVEL SECURITY`，而 Supabase 默认会给 public schema 新表授予 `anon` 全部权限。前端的 publishable key 是公开的（打进浏览器 bundle，任何人都能拿到）。

**实测证据**（在测试项目 `rlakqwmedlgpcdasvads` 上，仅操作自建测试行，已清理）：

| 操作（仅用公开的 publishable key） | 结果 |
| --- | --- |
| 读 `timer_sessions` | **成功**，读到 `customer_name`、`billing_minutes`、`amount_gbp`、`actual_amount_gbp`、`settlement_note` |
| 改写 `timer_sessions` | **成功**（把顾客姓名改成 `ANON-HACKED`，用 secret key 复核确认真的落库） |
| 删除 `timer_sessions` | **成功**（测试行被删掉，复核为空） |
| 读 `blocked_time_slots` | **成功** |
| 删除 `blocked_time_slots` | **成功** |

对照组（都正常，说明 RLS 本身有效）：`bookings`、`booking_events`、`processed_webhook_events`、`gallery_items`、`slot_templates` 匿名读不到；`tables` 匿名可读（001/002 里本来就是公开读，符合预期）；**`coupons` 匿名读不到**（013 已启用 RLS，本次改动是安全的）。

**影响**：掌握前端 bundle 里那把公开 key 的任何人都能读取全部顾客姓名、计时、账单和结算备注，并且能任意篡改或删除计时订单数据。不需要登录后台，也不需要任何权限。

**最小修复方案（已提供为 `supabase/migrations/014_enable_rls.sql`）**：只含两行 `ALTER TABLE ... ENABLE ROW LEVEL SECURITY`，**不建任何策略**（即 anon / authenticated 完全无权访问）。本次已逐个文件核对过，全部代码路径（后台接口、顾客自助接口、顾客端页面、lib 里的工具函数）访问这两张表时用的都是 `createAdminClient()`（service role，天然绕过 RLS），所以这个改动对应用零影响；Supabase Studio 用 service role，也不受影响。

该迁移与优惠券改动**分开提交**，便于维护方单独取舍。应用后的复验结果见下方。

**复验结果（已在测试项目 `rlakqwmedlgpcdasvads` 上实测）**：

| 攻击（仅用公开的 publishable key） | 修复前 | 修复后 |
| --- | --- | --- |
| 读 `timer_sessions` | 读到顾客姓名、账单、备注 | **0 条**（对照组：secret key 可见 1 条，数据确实存在） |
| 改写顾客姓名 | 成功落库 | 未命中任何行，复核姓名未变 |
| 篡改账单金额 / 结算状态 | 可写 | 复核金额仍为 25.99、状态仍为已结算 |
| 删除 `timer_sessions` 订单 | 成功删除 | 复核订单仍在 |
| 读 / 删 `blocked_time_slots` | 成功 | 读 0 条、删除未生效 |
| 读 `bookings` / `booking_events` / `processed_webhook_events` / `coupons` | 已被 RLS 挡住 | 依然挡住 |

同时用 secret key 重跑了全套 80 项断言，**全部通过**，应用侧行为零变化（service role 天然绕过 RLS）。

**生产库是否同样暴露尚未确认**：迁移脚本定义的状态是未启用，但维护方可能手工在 Dashboard 里开过。可在 Supabase Dashboard → Table Editor 看这两张表是否带 RLS 标记，或者用一个只读探测确认（见对话记录）。
