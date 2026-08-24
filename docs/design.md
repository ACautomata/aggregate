# 系统设计 — L3 盲评程序

面向 BraTS L3 终验「专家目检验收」的盲评前后端：≥2 位神经放射学评审者在 web 界面完成视觉图灵（real/synth）+ Likert 4 维评分，产出 `brats-l3-responses/1`，供 `scripts.brats_l3_blind_eval aggregate` 消费并解锁 #58 的 `conclude` / `dm_source` 注册。

设计前提与契约以 NV-Generate-CTMR 仓库的 `scripts/brats_l3_blind_eval.py`（stdlib、纯 python）为权威，本仓库不做重复实现，仅对接。术语见 `CONTEXT.md`，硬决策见 `docs/adr/0001` 与 `0002`。

## 1 架构总览

```
┌──────────────────────── 后端服务器（独立于 gauss） ────────────────────────┐
│  评审者浏览器 ──HTTPS/内网──▶  Fastify(TS)                                 │
│                               ├─ 静态托管: 构建后的前端 SPA (Vue3)          │
│                               ├─ API 层 (zod 校验)                         │
│                               │    session / package / image /             │
│                               │    progress / answer / submit / export     │
│                               ├─ 持久化: 后端服务器本地 storage            │
│                               │    responses/  progress/  (非数据本体)      │
│                               └─ 渲染器: 3D→2D 切片 + 窗宽窗位 → PNG         │
└───────────────┬─────────────────────────────────────────────────────────┘
                │  tailscale（首选）│ ssh 隧道（备选）
                ▼
   ┌────────────────────────── gauss（唯一数据库）────────────────────────┐
   │   PostgreSQL 16（数据本体 / 只读入口）                                │
   │   images(bytea)   package/bindmap/catalog(JSONB)   reviewers          │
   └─────────────────────────────────────────────────────────────────────┘
```

数据流（sugon → gauss → 评审 → 聚合）：

```
sugon 受控产物 ──(rsync 全量)──▶ gauss 数据本体(PG)
                                      │
                匿名化：评审侧只见 {entry_id, challenge, target_modality} + 匿名影像
                                      ▼
            评审者输入身份码 → 逐条评分(3D 浏览+W/L) → 校验 → responses/<code>.json
                                      │
        python3 -m scripts.brats_l3_blind_eval aggregate --run <run.json> \
            --responses R1.json --responses R2.json --blind-map blind_map.json \
            --catalog catalog.json --output l3_report.json --resamples 1000 --seed 20260821
```

## 2 数据模型（PostgreSQL · DDL 级）

> 受控字段标记为 **[受控]**：存在数据本体，任何 API 响应与前端界面不得携带。

```sql
-- 清单文档（原样整存，可导出回文件喂 aggregate）
CREATE TABLE package_doc ( id TEXT PRIMARY KEY, doc JSONB NOT NULL );   -- brats-l3-package/1
CREATE TABLE blind_map_doc ( id TEXT PRIMARY KEY, doc JSONB NOT NULL );  -- brats-l3-blind-map/1
CREATE TABLE catalog_doc  ( id TEXT PRIMARY KEY, doc JSONB NOT NULL );   -- brats-l3-catalog/1

-- 盲评条目：每 API 响应一条（影像 + 受控真值）
CREATE TABLE blind_entries (
    entry_id        TEXT PRIMARY KEY,          -- L3-0001..L3-0200
    challenge       TEXT NOT NULL,             -- GLI/SSA/MEN/METS/PED
    target_modality TEXT NOT NULL,             -- t1n/t1c/t2w/t2f
    src_modality    TEXT,                      -- P3 可选，保留
    source          TEXT NOT NULL,             -- real|synth   [受控]
    case            TEXT NOT NULL,             -- 病例 ID       [受控]
    image_bytea     BYTEA NOT NULL,            -- 原始 .nii.gz 字节流
    image_sha256    TEXT NOT NULL,             -- = catalog.json sha256（对账）
    meta            JSONB NOT NULL             -- shape/spacing/dtype 等渲染所需
);

-- 身份识别码：一评审者一码，码即 reviewer 标识
CREATE TABLE reviewers (
    code        TEXT PRIMARY KEY,              -- 8 位 crypto 随机，去易混字符
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at  TIMESTAMPTZ,                   -- NULL=有效；无自动过期
    note        TEXT
);

-- 评审响应：草稿与已提交同表，靠 status 区分；PK 强约束「每码每条目恰一 slot」
CREATE TABLE responses (
    reviewer    TEXT NOT NULL REFERENCES reviewers(code),
    entry_id    TEXT NOT NULL REFERENCES blind_entries(entry_id),
    status      TEXT NOT NULL DEFAULT 'draft', -- draft|submitted
    turing      TEXT CHECK (turing IN ('real','synth')),
    overall_realism            SMALLINT CHECK (overall_realism BETWEEN 1 AND 5),
    anatomical_plausibility     SMALLINT CHECK (anatomical_plausibility BETWEEN 1 AND 5),
    tumor_authenticity          SMALLINT CHECK (tumor_authenticity BETWEEN 1 AND 5),
    artifact_slice_consistency  SMALLINT CHECK (artifact_slice_consistency BETWEEN 1 AND 5),
    likert_na_flags             JSONB,         -- 记录该条哪些维为 NA（值域以 zod 为准）
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (reviewer, entry_id)
);
CREATE INDEX idx_responses_reviewer ON responses(reviewer);

-- 唯一索引：每评审者至多一条 submitted 记录（覆盖恰一次）—— 兜底
CREATE UNIQUE INDEX one_submission_per_reviewer
    ON responses(reviewer) WHERE status='submitted';
```

**设计要点**：

- 影像按**原始字节流**入库，元数据单列 `meta`（JSONB），不预解包体素——权威副本，`image_sha256` 可对账；避免后端/catalog 不一致。
- `responses` 以 `(reviewer, entry_id)` 为主键，DB 层强制「每码每条目恰一个槽位」，配合 zod 校验构成双重防线。
- 清单文档以 JSONB 整存，`aggregate` 需要 `--blind-map` 与 `--catalog` 文件时直接从 DB 导出（详见 §7 部署）。
- Likert 值域 `1..5 | NA`：DB `CHECK` 兜底，**zod 是前置校验**——python 聚合器 `_validate_responses` 只校验 `turing` 与覆盖，不校验 Likert 值域，缺省校验由本程序承担（否则聚合期以 `TypeError` 而非干净的 `L3Error` 失败）。

## 3 后端 API 契约（Fastify + zod）

所有请求/响应均 JSON；`[受控]` 字段绝不出现在任何响应。

### 会话

| 方法 | 路径 | 说明 |
|---|---|---|
| `POST` | `/api/session` | 入参 `{code}`；校验身份码（存在且未注销），返回 `{reviewer, package_meta}`。失败 401。页面加载即调用，码即 session 凭据 |

### 包与影像

| 方法 | 路径 | 说明 |
|---|---|---|
| `GET` | `/api/package` | 匿名化条目列表：每项 `{entry_id, challenge, target_modality, slices:{axial,coronal,sagittal}}`（切片数供前端滑窗）。**不含** `source/case/image_path` |
| `GET` | `/api/image/:entry_id` | 渲染 2D 图。query：`view=axial\|coronal\|sagittal`、`slice=<n>`、`wl=<int>`、`ww=<int>`；响应 PNG。后端自 3D 字节流转 2D 并做窗宽窗位。LRU 缓存 + 请求防抖 |
| `GET` | `/api/image/:entry_id/meta` | 该条目 `{shape, spacing, dtype, views}`，前端初始化视口与滑窗范围、默认窗 |

### 评审（草稿与提交）

| 方法 | 路径 | 说明 |
|---|---|---|
| `GET` | `/api/progress/:reviewer` | 已填答案 `{entry_id: {turing, likert...}}`；续填/换浏览器恢复 |
| `PUT` | `/api/answer/:reviewer/:entry_id` | 保存单条：`{turing, overall_realism, anatomical_plausibility, tumor_authenticity, artifact_slice_consistency}`，Likert 允许 `'NA'`；写 `status='draft'` |
| `POST` | `/api/submit/:reviewer` | 校验：① 覆盖全部 200 个 `entry_id` 恰一次；② 每维 `1..5` 或 `NA`；③ `turing ∈ {real,synth}`；通过后 `status='submitted'` 原子落盘 |
| `GET` | `/api/response/:reviewer/download` | 导出完整 `brats-l3-responses/1` JSON（校验通过后才可用），即 aggregate 的 `--responses` 输入 |

### zod 关键模式（前端+后端共用类型，`src/shared/schemas.ts`）

```ts
const TURING = z.enum(['real', 'synth']);
const LIKERT = z.union([z.number().int().min(1).max(5), z.literal('NA')]);
const Dimensions = z.object({
  overall_realism: LIKERT, anatomical_plausibility: LIKERT,
  tumor_authenticity: LIKERT, artifact_slice_consistency: LIKERT,
});
const Rating = Dimensions.extend({ turing: TURING });
const Response = z.object({
  schema: z.literal('brats-l3-responses/1'),
  reviewer: z.string(),                       // = 身份识别码
  entries: z.array(Rating.extend({ entry_id: z.string() })).min(200).max(200),
});
```

## 4 影像浏览器（评审界面交互）

- **视图**：三向 MPR（轴位 / 冠位 / 矢位）层滑块，外加窗宽窗位（滑动）与预设窗（脑组织、骨、宽窗）。不做 MIP、测量、标注（保简易）。
- **渲染路径**：浏览器内不做像素窗——请求 `/api/image/:entry_id?view=&slice=&wl=&ww=`，后端读数据本体字节流，nifti 解析 → 取切片 → 按 `wl/ww` 线性窗映射到 8bit → LUT → PNG。防抖合并拖动，往返在 tailscale/内网延迟下可接受。此路径将"3D→2D"留在后端（符合需求 3「后端转换」），API 已参数化，将来可无缝切换为前端本地 windowing（后端只送 16bit 原始切片）。
- **每条交互**：左大图 + 视图/层/窗控件；右侧 `turing` 单选 + 4 个 Likert 下拉（含 `NA`）。完成一条即 `PUT /api/answer` 存草稿。
- **进度与续填**：答案以 `(code, entry_id)` 存服务端 `responses` 表；换浏览器/中断后可恢复。可选 localStorage 镜像。
- **提交**：全部 200 条填完（校验恰一次 + 值域）后提交 → 后端原子写 `responses/<code>.json` 并置 `submitted` → 页面提示导出/完成。

## 5 身份识别码机制

- **生成**：管理方 CLI（`node scripts/generate-codes.mjs <count>`）用 `crypto.randomBytes` 生成 8 位码，剔除易混字符（如 `0/O`、`1/l/I`），一码一人，落入 `reviewers` 表。
- **语义**：码即 reviewer 标识——聚合报告的 `reviewer` 字段=码；匿名、可审计、不与真实身份绑定。无自动过期，admin 手动注销（`revoked_at`）或重发。
- **会话约定**：一码一会话；同一码不允许多端同时作答。前端输入码 → `POST /api/session` 校验 → 进入。
- **DUA 边界**：码不关联任何病例/受试者信息；管理侧知道的只是"评审者 X 已完成 blind-eval"。

## 6 DUA 与匿名化

- 评审者可感知字段白名单：`entry_id`（L3-XXXX）、`challenge`、`target_modality`、匿名影像。
- **永不出现**：`case`（病例 ID）、`subject ID`、`split` 划分信息、来自 `image_path` 的任何含 case 的线索、`source`（real/synth 真值）。
- 匿名化在**数据迁移时完成**（sugon 侧拷贝+重命名为 `L3-XXXX.nii.gz`），入库即匿名；渲染不再引入病例标识。
- 后端 API 层对 `blind_entries` 投影为「评审可见字段」；`source/case/image_bytea` 仅内部渲染与聚合导出使用。
- 报告（`l3_report`）与 `blind_map` 只在受控存储/数据库，不入 Git（契约 DUA 校验拒绝 Git 内报告）。

## 7 与聚合脚本的对接

- 产物：`blind-map`、`catalog`、`responses/<code>.json` 由数据库导出为文件（或直接路径引用）。
- 命令（`aggregate` 要求 ≥2 份独立响应、覆盖全部 200 条目、Likert `NA` 的 cell 无任何 score 时报错）：
  ```sh
  python3 -m scripts.brats_l3_blind_eval aggregate \
      --run <run.json> --responses R1.json --responses R2.json \
      --blind-map blind_map.json --catalog catalog.json \
      --output l3_report.json --resamples 1000 --seed 20260821
  ```
- 后端负责把 `responses` 表聚成单个 reviewer 的 `brats-l3-responses/1` 文件（含全部 200 条、已校验、无 NA 缺省冲突）。正确性：以 `scripts.brats_l3_blind_eval selftest` 为现成参照（fixture 驱动）。

## 8 技术栈清单

| 层 | 选型 |
|---|---|
| 后端 | Node 22 + Fastify 5 + zod（类型/校验与前端共享）+ pino 日志 + `@fastify/static`（托管前端）+ `@fastify/cors`（如需跨域）|
| DB 驱动 | `postgres`（porsager/postgres.js）或 `pg`——偏好 postgres.js（TS 友好、单文件）|
| 影像解析 | `nifti-reader-js`（Node 解析 .nii.gz 字节流 → int16 矩阵）|
| PNG 编码 | `sharp`（raw int16 → windowed 8bit → PNG）|
| 前端 | Vite + Vue 3 + TS；canvas 自定义渲染（仅显示 PNG；不 dwn 像素处理）|
| UI | Vue 单页：身份码入口 → 条目列表 → 条目评分页（MPR + W/L + 表单）|
| 部署 | Node 静态 tarball（非 root，`/data72/junran/opt/node`）+ pm2；gauss 侧 PostgreSQL 16（用户目录 initdb）|
