# CONTEXT — L3 盲评程序

单一上下文。本仓库（aggregate）承载面向 BraTS L3 终验盲评的前后端程序设计与文档；数据/契约事实以 NV-Generate-CTMR 仓库（`scripts/brats_l3_blind_eval.py`、`brats_phase_run_contract.py`）为准，不重复登记。

## Language

### 盲评数据

**数据本体 (data body)**:
承载盲评权威数据（NIfTI 3D 卷与 package/catalog/blind-map 清单）的存储体——部署在 gauss 服务器的**唯一数据库**中，与后端程序及其持久化物理/逻辑分离；后端仅通过数据库接口访问它。
_Avoid_: 把评审响应当作数据本体、把数据本体当作后端本地文件

**后端持久化 (backend persistence)**:
后端程序自有、随其部署的存储（评审响应、评审进度），不属于数据本体，不进入 gauss 数据库。
_Avoid_: 将响应写入数据本体目录、把数据库当后端缓存

**匿名化 (anonymization)**:
消除病例/受试者/划分标识（case ID、subject ID、split 信息）的副本过程——影像以盲评条目号 L3-XXXX 命名，容器内不出现病例标识。
_Avoid_: 直接把含 case 路径的原始文件暴露给评审侧

**盲评条目 (blinded entry)**:
评审的最小单元，编号 L3-0001..L3-0200；向评审者只暴露 {entry_id、子挑战、目标模态} 与匿名影像。
_Avoid_: source 真值（real/synth）、case ID、原始路径

**盲映照表 (blind map)**:
entry_id ↔ {source、病例、真实路径} 的受控映射（schema `brats-l3-blind-map/1`），仅数据本体持有，永不出现在评审界面与后端 API 响应中。

### 评审

**评审者身份识别码 (reviewer access code)**:
管理方生成、评审者在界面输入后进入评审系统的门禁码；一份响应以该码归属，码即 reviewer 标识。
_Avoid_: 把身份识别码当作病例/人员真实身份、评审者间共享

**评审响应 (review response)**:
一位评审者对全部 200 条盲评条目的判定集（schema `brats-l3-responses/1`：turing ∈ {real,synth}，Likert 1..5 或 NA），由后端校验后持久化，供 `brats_l3_blind_eval aggregate` 消费。
_Avoid_: 部分条目响应、重复条目、turing/Likert 缺省

**影像浏览器 (image browser)**:
评审界面中浏览盲评条目的交互组件——切片（层滑动、多平面）与窗宽窗位调节；2D 图像由后端自 3D 数据转换。

### 部署

**唯一数据库 (the single database)**:
gauss 服务器上承载数据本体的 PostgreSQL 实例（2026-08-24 定）；后端服务器经 tailscale（首选）或 ssh 隧道（备选）访问之。
_Avoid_: 多实例各存一份影像导致不一致
