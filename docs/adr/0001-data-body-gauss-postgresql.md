# ADR-0001：盲评数据本体= gauss 唯一 PostgreSQL

- **状态**：accepted（2026-08-24）
- **范围**：L3 盲评程序（本仓库）中，需求点 1「数据保存在 gauss」与需求点 3「数据与后端持久化分离」的落地方式。

## 背景

sugon 上已生成盲评自动化产物（`package.json` 200 条、`blind_map.json`、`catalog.json`）与合成/真实影像，均在受控目录。用户要求：① 数据保存在 gauss；② 数据从 sugon 传回 gauss（在实现过程中）；③ 数据与后端持久化分离。gauss 无 node（python 仅 3.8），有 `/data72`（余 36T）与既有数据目录惯例（`/data72/dataset/`、`/data72/junran/<project>`、`<project>-runtime`）。

## 决定

数据本体（盲评权威数据：NIfTI 3D 卷 + 三份清单文档）落户 **gauss 服务器上的唯一 PostgreSQL 实例**，与后端程序及其持久化物理/逻辑分离。后端是唯一访问本体的程序，仅经数据库接口读取。

- 影像以**原始 `.nii.gz` 字节流**（`BYTEA`）入库，另存元数据列（形状、spacing、sha256），不预解包体素——原文即权威副本，`catalog.json` 的 sha256 可直接对账。
- 三份清单（`brats-l3-package/1`、`brats-l3-blind-map/1`、`brats-l3-catalog/1`）以 **`JSONB`** 文档形式整存，可原样导出回文件供 `brats_l3_blind_eval aggregate` 消费。
- 后端持久化（响应、评审进度）存在于后端服务器自身存储，**不**写入数据本体。

选用 PostgreSQL 而非其他：BYTEA 存二进制、JSONB 存清单文档、SQL 约束保证「响应每条目恰一次」与值域完整性，单机 6GB 级影像无性能压力。

## 考虑过的替代

- **MongoDB**：亦能存二进制+文档，但缺少跨表 join 引用约束与 SQL 审计。
- **只读文件目录**（最初提案）：权威、零组件、可直接对接 aggregate；被用户「部署数据库、影像入库、后端走 DB 接口」的方案覆盖。

## 后果

- gauss 需一个可运行、监听 tailscale 接口、受 `pg_hba` 限制的 PostgreSQL 实例（非 root，用户目录 `initdb` 跑）。
- 后端经 tailscale（首选）或 ssh 隧道（备选）连接 gauss PG，凭据不入仓库。
- 受控字段（`source`、`case`、`image_path`）入库但不经 API 暴露，见 ADR-0002。
