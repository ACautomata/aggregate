# 部署与数据迁移手册 — L3 盲评程序

> 说明：本手册交付**文档**，不实际执行。所有命令以 gauss / sugon / 后端服务器的实际环境为准（gauss 经 `junran@gauss`（172.20.114.162，经 euler 跳板）；sugon `root@ssh.zzai.scnet.cn:10112`）。**后端服务器未定**，凡依赖它的路径以占位符 `<BACKEND_HOST>`、`<BACKEND_PORT>` 表示，具体机组确定后填实。

## 0 拓扑回顾

```
评审者浏览器 → [后端服务器 <BACKEND_HOST>:<BACKEND_PORT>] ——tailscale/ssh──▶ [gauss: PostgreSQL 数据本体]
                                                    └── 数据迁移(一次性)：sugon → gauss
```

三段接入：
1. **数据迁移**（sugon→gauss）：一次性，迁移脚本临时使用、不入仓库。
2. **后端服务器→gauss**：tailscale（首选）或 ssh 隧道（备选）。
3. **评审者→后端服务器**：模式 A 公网 HTTPS（Caddy 反代）或模式 B 内网直连（`<BACKEND_HOST>` 与 gauss 同内网）。模式未定，先按下述两分支准备。

## 1 gauss：初始化 PostgreSQL（数据本体）

gauss 无 root（sudo 需密码）、python 3.8、无 node，但 `/data72` 余 36T。用**用户目录 initdb** 的方式在非 root 下跑 PG 16。

```sh
# 以 junran@ gauss 执行
APT_ARCHIVE=https://ftp.postgresql.org/pub/source  # 或发行版自带 postgresql-16 二进制包；非 root 亦可源码编译
PGSRC=~/opt/postgresql-16
mkdir -p ~/opt ~/brats-l3
cd ~/opt && tar xf postgresql-16*.tar.bz2 && cd postgresql-16 && ./configure --prefix=$PGSRC && make -j4 && make install

# 初始化数据目录（数据本体即此 PG 实例）
export PATH=$PGSRC/bin:$PATH
initdb -D ~/brats-l3/pgdata -U blindapp --auth=scram-sha-256

# 只监听 tailscale / 内网接口，并建库
cat >> ~/brats-l3/pgdata/postgresql.conf <<EOF
listen_addresses = '<GAUSS_TS_IP>'       # tailscale 或内网 IP；0.0.0.0 亦可但需 pg_hba 收紧
port = 5432
EOF
pg_ctl -D ~/brats-l3/pgdata start
createdb -U blindapp brats_l3
```

> **安全**：`pg_hba.conf` 只放行后端服务器（tailscale 网段 `100.64.0.0/10` 或按实际网段）；绝不开放公网 5432。凭据（连接串 `postgres://blindapp:<pwd>@<GAUSS_TS_IP>:5432/brats_l3`）不进仓库——后端用环境变量注入。

## 2 数据迁移（sugon → gauss，一次性）

### 2.1 匿名化（在 sugon 上做，先于传输）

`package.json` 的 `image_path` 当前含 case ID（如 `BraTS-GLI-01308-000_t1c_seed....nii.gz`），评审侧不得泄露。需在 sugon 侧**拷贝 + 重命名**为 `L3-XXXX.nii.gz`（无 case 标识），并解析出真实/合成卷。

```sh
# sugon 上：匿名化 200 条（100 合成 + 100 真实），输出到 /root/anonymized-l3/
#   package.json 每条 entry_id L3-XXXX ↔ image_path（合成=generated/<CH>/<case>/，真实=rflow_phase/raw）
#   读 package.json 的 entry_id 与 image_path，逐条 cp 为 /root/anonymized-l3/images/L3-XXXX.nii.gz
#   只取 200 条（合成 100 卷 + 真实对应 100 卷），不搬整个 6.4G 生成目录
```

匿名化脚本本轮以**文档形式**提供（临时使用、不入仓库）。要点：
- 对每条 `package.json` 的 `entry_id`，取其 `image_path`，拷贝为 `L3-XXXX.nii.gz`（文件名不含 case）。
- 真实侧卷自 `brats2023_rflow_phase/raw/...`；合成侧自 `holdout_generated/generated/<CH>/<case>/`。
- 计算每卷 sha256，用于与 `catalog.json` 对账。

### 2.2 传输（rsync 增量）

```sh
# 本机（或后端服务器）执行：sugon → gauss
rsync -avzP --info=stats1 \
  -e "ssh -p 10112" \
  root@ssh.zzai.scnet.cn:/root/anonymized-l3/ \
  <BACKEND-or-transfer-host>:/tmp/anonymized-l3/   # 先到中转，再进 gauss

# 中转 → gauss（经 euler 跳板）
rsync -avzP -e "ssh" /tmp/anonymized-l3/ junran@gauss:~/brats-l3/incoming/
```

> 若「后端服务器」即中转，可直接 `rsync sugon → <BACKEND_HOST>:incoming` 再 `→ gauss`。文档不强绑；聚合需要的是**文件最终落到 gauss 数据本体**。

### 2.3 入库（gauss 上导入到 PG）

```sh
# junran@gauss：将清单与影像灌入数据本体
psql "postgres://blindapp:<pwd>@<GAUSS_TS_IP>:5432/brats_l3" <<SQL
-- package_doc / blind_map_doc / catalog_doc 以 JSONB 整存（由 json 文件 COPY 或脚本写入）
COPY package_doc(id,doc)  FROM '/home/junran/brats-l3/incoming/package.json'   ...
COPY blind_map_doc(id,doc) FROM '/home/junran/brats-l3/incoming/blind_map.json' ...
COPY catalog_doc(id,doc)  FROM '/home/junran/brats-l3/incoming/catalog.json'   ...

-- blind_entries：展开 package + blind_map + catalog，并读入影像字节
INSERT INTO blind_entries(entry_id,challenge,target_modality,src_modality,source,case,image_bytea,image_sha256,meta)
SELECT ... FROM blind_map_doc, catalog_doc, pg_read_binary_file('incoming/images/'||entry_id||'.nii.gz');
SQL
```

> 敏感：`incoming/` 含受控字段与影像，导入完成后清理该目录（DUA）。`blind_map` 永不导出到评审可访问路径。

## 3 后端服务器：node + pm2 + 应用

```sh
# 以非 root 用户，在 <BACKEND_HOST>
# node 静态 tarball（与本地一致 v22）
mkdir -p ~/opt ~/opt/node-tarball && tar xJf node-v22*.tar.xz -C ~/opt
export PATH=$HOME/opt/node-v22*/bin:$PATH && node -v
npm i -g pm2

git clone <该仓库> ~/aggregate && cd ~/aggregate
pnpm install && pnpm build                 # 构建前端 SPA 到 dist/
# 配置：.env 注入数据库连接串（走 gauss tailscale IP/端口）+ <BACKEND_PORT>
pm2 start dist/server/index.js --name blind-eval
pm2 save && pm2 startup                      # 免 root 自启
```

**gauss 连接（tailscale 首选 / ssh 隧道备选）**：

```sh
# 首选：backend 装入 tailscale 并 join 到 gauss 所在 tailnet
# 后端服务器 tailscale up --login-server ... ; 两人同 tailnet → 直连 <GAUSS_TS_IP>

# 备选：ssh 隧道（不依赖 tailscale）
ssh -N -L 5432:localhost:5432 -o ServerAliveInterval=60 junran@gauss &
```

## 4 评审者访问（模式 A / B）

### 模式 A：公网 HTTPS（Caddy 反代）
```sh
# 后端服务器
sudo caddy reverse-proxy \
  --from https://blind.example.com \
  --to localhost:<BACKEND_PORT>              # 自动 Let's Encrypt
```
前端与 API 同源，`@fastify/static` 托管 `dist/`；无 CORS 配置需求。

### 模式 B：内网直连
```sh
# 后端服务器监听内网地址；评审者内网访问 http://<BACKEND_HOST>:<BACKEND_PORT>
# 若 `npm audit` 无涉，可直接不加代理；如需 TLS，信任内网证书即可
```

## 5 评审与管理员操作手册

### 生成身份识别码
```sh
node scripts/generate-codes.mjs 3        # 生成 3 个独立 8 位码，写入 reviewers 表
```
码即 reviewer 标识（一码一人一会话）；无自动过期，管理方可注销/重发。

### 评审者流程
1. 打开界面 → 输入自己的身份识别码 → `POST /api/session` 校验。
2. 逐条：MPR 三向浏览 + 调窗宽窗位 → `turing` real/synth → 4 维 Likert（1–5 或 NA）→ 保存草稿。
3. 填完 200 条 → 提交（后端校验覆盖恰一次 + 值域）→ 导出 `responses/<code>.json`。

### 完成评审后聚合
```sh
# 从数据库导出 blind_map.json / catalog.json / responses/<code>.json（受控）
# 然后（在 gauss 或后端，python3 环境，仓库 scripts/）
python3 -m scripts.brats_l3_blind_eval aggregate \
    --run <run.json> --responses R1.json --responses R2.json \
    --blind-map blind_map.json --catalog catalog.json \
    --output l3_report.json --resamples 1000 --seed 20260821
# → l3_report.json；再走契约 attach l3_report → conclude → dm_source 注册
```

## 6 备份与审计

- **数据本体**：`pg_dump brats_l3`（含 JSONB + BYTEA）定期备份到 gauss 受控目录；影像可另以 `rsync` 保留 `incoming/` 原始匿名卷的副本。
- **响应**：后端 `responses/` 目录 `rsync` 到受控存储；提交后即为 immutable 审计件。
- **演进迁移**：PG dump 可移植；改名/换 DB 后端只改连接串（env）。

## 7 未决项 / 待确认

| 项 | 状态 | 影响 |
|---|---|---|
| 后端服务器具体机组 | 未定 | `<BACKEND_HOST>`、`<BACKEND_PORT>` 占位；模式 A/B 待选 |
| 评审者网络 | 未定 | 决定模式 A（需公网）或 B（需与 gauss 同内网）|
| gauss PG 版本 | 待定 | 建议 16；若发行版只有 14/15 亦可（子集兼容）|
| 是否放开 5432 到内网 | 待定（建议收紧） | pg_hba 只放 tailscale 网段 |

> 所有受控数据（影像、blind_map、报告）保持在受控存储/数据库，不入 Git。
