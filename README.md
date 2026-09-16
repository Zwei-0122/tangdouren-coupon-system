# Tangdouren 优惠券系统 · 交付包

基准提交：`fa00456`（交付时的 `origin/main` 顶端）
交付分支：`feat/coupon-system`，共 4 笔提交

| 提交 | 内容 |
| --- | --- |
| `b77b58f` | 优惠券系统本体（PRD v1.0 全量实现，11 个文件，迁移 013） |
| `c3dea24` | 安全修复：为 `timer_sessions`、`blocked_time_slots` 启用 RLS（迁移 014） |
| `acda6e4` | 活动前缀功能（PRD v1.1 变更，6 个文件，迁移 015） |
| `1778db4` | 后台侧边栏把「优惠券」入口移到列表末尾 |

相对基线合计 13 个文件，+2180 / −59。

## 包内文件

| 文件 | 说明 |
| --- | --- |
| `patches/0001-feat-add-coupon-generation-and-atomic-redemption-for.patch` | 优惠券系统本体，11 个文件 |
| `patches/0002-fix-enable-RLS-on-timer_sessions-and-blocked_time_sl.patch` | 安全修复：为 `timer_sessions` 与 `blocked_time_slots` 启用 RLS |
| `patches/0003-feat-add-campaign-code-prefix-to-coupon-generation.patch` | 活动前缀：迁移 015 + 前缀校验与生成、接口、后台页面、CSV、测试 |
| `patches/0004-feat-move-coupon-nav-entry-to-bottom-of-admin-sideba.patch` | 侧边栏优惠券入口位置 |
| `Tangdouren-coupon-system.bundle` | 4 个提交的增量 git bundle（可选） |
| `Tangdouren-优惠券系统-交付说明.md` | 完整交付文档：改动清单、迁移执行方法、测试结果、与 PRD 的差异、风险与安全发现 |

## 怎么应用

### 方式一：patch（推荐，不需要仓库权限）

```bash
git checkout -b feat/coupon-system fa00456
git am /path/to/patches/*.patch        # 0001 → 0004，按文件名顺序
```

已实测：四个补丁按顺序干净应用，应用后的文件树与本交付分支完全一致（文件树对象 `647bc9f7`，134 个文件一致）。

### 方式二：bundle

```bash
git fetch /path/to/Tangdouren-coupon-system.bundle feat/coupon-system:coupon-system
```

bundle 为增量包，需要目标仓库已有 `fa00456`。实测拉出来的分支与本交付分支文件树一致。

### 方式三：只要安全修复

如果暂时不想合代码，只想先把漏洞堵上，单独执行 `0002` 里的迁移即可（就两行）：

```sql
ALTER TABLE timer_sessions     ENABLE ROW LEVEL SECURITY;
ALTER TABLE blocked_time_slots ENABLE ROW LEVEL SECURITY;
```

## 数据库迁移

1. `supabase/migrations/013_coupons.sql`：优惠券表、约束、索引、RLS、结算核销与撤销两个事务函数
2. `supabase/migrations/014_enable_rls.sql`：安全修复
3. `supabase/migrations/015_coupon_prefix.sql`：`coupons.code_prefix` 列 + 格式与一致性 CHECK + 索引 + 前缀用量统计函数

三者都是纯增量、向前兼容的变更，可在现有 Supabase 项目直接执行，执行方法见交付说明第 2 节。

**执行顺序：`015` 必须在 `013` 之后**（它依赖 `coupons` 表，CHECK 还会校验 `code_prefix` 与 `code` 开头一致）。**接手的人注意：不跑 `015` 就生成优惠券，会直接报缺列。**

## 产品行为上需要知道的四件事

1. **取消手动改价**：结算金额全部由服务端计算，页面不再有实收金额输入框，客户端传入的金额一律忽略。
2. **撤销结算不回退预约状态**：关联预约仍保持「已完成」，需要人工在预约管理页处理。
3. **时长券按半小时续时价统一抵扣**：30 分钟券 = £6.00、60 分钟券 = £11.99，与订单时长无关；首小时不可减免。
4. **优惠码前缀可自定义**：店员按活动填前缀（如 `WELCOME`、`LUCKY`），2 到 8 位大写字母数字，留空回退默认 `TD`；后缀仍由系统随机生成，5 位。规则全表见交付说明第 5.1 节。

更完整的差异说明、测试结果与风险清单见交付说明书。
