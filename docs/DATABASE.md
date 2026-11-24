# 数据库与筹码账目设计

## 1. 原则与运行配置

SQLite 是永久事实源，PDO 参数绑定，UTC 时间以 ISO-8601/整数毫秒存储。每个连接启用 `PRAGMA foreign_keys=ON`；生产采用 WAL、`synchronous=FULL`（性能验收后才可调整）、`busy_timeout=5000`。写事务短小，`SQLITE_BUSY` 采用带抖动的有界退避，总计约 10 秒后返回可重试错误，不能无限等待。

迁移使用单调版本、校验和及 `schema_migrations(version, name, checksum, applied_at)`；应用启动先验证版本，不自动执行未经运维确认的破坏性迁移。Phase 1A/1A.1 不创建数据库或运行迁移。

余额性质金额使用 SQLite INTEGER，并加 `CHECK(value BETWEEN 0 AND 1000000000000)`；有符号流水增量和净盈亏允许负数，但绝对值及所有中间结果也不得超过 `1_000_000_000_000`。目标 PHP 必须为 64 位；该上限能被 PHP int、SQLite INTEGER 和 JavaScript Number 精确表示。ID 使用不可猜测字符串/UUID。

## 2. 表结构草案

### 身份与配置

| 表 | 关键字段/约束 |
|---|---|
| `accounts` | `id PK`, `username UNIQUE COLLATE NOCASE`, `password_hash`, `role CHECK(admin/player)`, `status`, `must_change_password`, timestamps |
| `players` | `id PK`, `kind CHECK(human/bot)`, `account_id UNIQUE NULL FK`, `display_name`, avatar, `enabled`; human 必须关联账号、bot 必须为空（触发器/应用双重保证） |
| `guest_players` | `id PK`, `guest_session_id UNIQUE`, display_name, `expires_at`; 不关联 wallet |
| `bot_profiles` | `player_id PK FK`, personality, difficulty, strategy_version, JSON 参数、约束版本；仅 bot |
| `sessions` | `id PK`, `account_id FK`, `token_hash UNIQUE`, issued/expires/last_seen/revoked_at, client metadata 摘要 |
| `invitations` | `id PK`, `code_hash UNIQUE`, `max_uses DEFAULT 1`, used_count, `expires_at`（默认创建后 7 天）, revoked_at/created_by；不存明文 |
| `system_settings` | `key PK`, JSON value, revision, updated_by/at；敏感密钥不存此表 |

### 房间、正式筹码与手牌

| 表 | 关键字段/约束 |
|---|---|
| `wallets` | `player_id PK FK`, `balance CHECK>=0`, `version`, updated_at；仅永久 human/bot |
| `rooms` | `id PK`, code UNIQUE, type normal/practice, `owner_account_id NULL FK`, `owner_guest_session_id NULL FK`, status, config_json, version；CHECK 恰有一个 owner；普通房必须 account，bot 不可为 owner |
| `room_seats` | `room_id, seat_no UNIQUE`, player/guest 二选一，status；普通房禁止 guest；对非空 player_id 建 `WHERE status IN (...)` 的部分唯一索引，保证全局最多一个有效座位 |
| `table_escrows` | `room_id, player_id` 复合主键，`balance CHECK>=0`, `version`; escrow 科目唯一定位房间与玩家，仅普通房 |
| `practice_stacks` | `room_id, participant_id UNIQUE`, `balance CHECK>=0`; 与 wallets/正式流水无 FK |
| `hands` | `id PK`, room_id, sequential_no UNIQUE/room, type, status STARTED/SETTLED/VOIDED, config_snapshot, button_seat, started/settled_at |
| `hand_checkpoints` | `hand_id PK`, participant_stacks_json, checksum, created_at；开手事务写入且不可改 |
| `hand_participants` | `hand_id, player_id/guest_id`, seat, opening_stack, gross_committed, uncalled_returned, net_committed, pot_awarded, final_stack, result；唯一约束 |
| `pots` / `pot_awards` | 每手各层金额、资格摘要；`hand_id,pot_no,player_id` 唯一，奖励非负 |
| `hand_settlements` | `hand_id UNIQUE`, settlement_key UNIQUE, totals/checksum, committed_at；幂等屏障 |

牌数据分为 `public_revealed_cards` 与 `private_hole_cards_encrypted`。有效正式手只为注册真人保留本人的加密私牌；应用层严格本人授权。机器人未公开牌、作废手私牌和无实际用途的剩余牌堆在结算/恢复后删除。管理员不因角色获得解密权限；但应用权限隔离不声称能够抵御拥有服务器 root 与密钥的操作者。密钥不进仓库或数据库。

### 流水、统计与审计

| 表 | 关键字段/约束 |
|---|---|
| `ledger_transactions` | `id PK`, `business_type`, `business_key UNIQUE`, actor, reason, created_at |
| `ledger_entries` | transaction_id, player_id, `room_id NULL`, account_kind wallet/escrow/system, `delta` 有符号整数, balance_after>=0；escrow 分录必须同时含 room_id |
| `player_stats` | player_id PK, hands_played/won/split, net_game_profit, max_win/loss, peak_assets, admin_grants/deductions, version |
| `hand_type_stats` | player_id, hand_category UNIQUE, count |
| `admin_audit_logs` | id, actor_account_id, action, target_type/id, before/after 安全摘要, request_id, ip 摘要, created_at；追加写 |
| `recovery_runs` | id, recovery_key UNIQUE, hand/room, action, result, audit payload, created_at |
| `request_dedup` | `principal_id, request_id` 复合唯一键, payload_hash, 经过该 principal 投影的 result_code/response_json, expires_at |

流水 delta 可为有符号，但余额、桌栈和池不得为负。管理员调整通过 system 发行/回收科目解释总量变化；游戏转移的所有分录净和必须为零。

邀请码注册使用单个 `BEGIN IMMEDIATE` 事务：按 code_hash 验证未撤销、未过期、used_count=0，条件更新为已使用，然后创建 Account、Human Player、初始钱包和初始流水；任一步失败全部回滚。唯一/条件更新保证并发只能成功一次。

## 3. 守恒式与事务

全局正式筹码：`Σ(wallet.balance) + Σ(table_escrow.balance)`。本手进行中 escrow 保持开手检查点余额，实时桌栈/投入/底池只是同一价值的内存拆分，资产统计不得重复相加。只有 `INITIAL_GRANT` 和 `ADMIN_ADJUSTMENT` 允许改变总量；练习筹码排除。

### 买入/追加

以 `(room_id, player_id, operation_id)` 为业务唯一键。`BEGIN IMMEDIATE` 后校验房间/边界、钱包版本和金额；钱包减 X、escrow 加 X、写一笔事务及两条分录，最后提交。重复业务键返回先前结果；余额不足整个事务回滚。追加只在两手间计入可用桌栈。

### 手牌结算

以全局唯一 hand_id/settlement_key 结算。对每人计算 `net_committed = gross_committed - uncalled_returned`，`net_game_profit = pot_awarded - net_committed = final_stack - opening_stack`；未跟注退还不计 pot_awarded。事务断言全体净盈亏之和为零、开手栈和等于终局栈和，再写池/奖励、最终 escrow、战绩和历史并设置 SETTLED。任何一步失败全回滚。

### 离桌返还

只能使用 escrow 当前余额，以唯一 leave operation 执行 wallet +X、escrow -X，并将座位离桌。若有活跃手牌，操作排队到结算/作废后；管理员不能绕过此边界直接编辑 escrow。

### 管理员调整

只改钱包，不改桌托管；请求含目标、正负金额、非空原因和 request_id。事务检查调整后非负、写 system 对手分录、更新管理员累计发放/扣除并追加审计。管理员调整自己使用相同流程。它改变资产但不改变游戏净盈亏。

## 4. 崩溃窗口与幂等恢复

1. **开手事务未提交**：不存在 STARTED/checkpoint，视为从未开手，无恢复动作。
2. **检查点已提交、结算未提交**：扫描到 STARTED 且无 settlement；以 `recovery:<hand_id>` 键将 escrow 恢复到 opening stacks，标 VOIDED，写恢复审计。该手不计战绩。
3. **结算事务提交成功、广播前崩溃**：存在 SETTLED/settlement；不得回滚或再次派奖。客户端重连从数据库快照得到结果。
4. **恢复后关闭旧房间**：对每位玩家用唯一返还键把 escrow 原子返还钱包；重复运行只读取既有结果。

恢复任务在实时服务接收新手牌前运行。状态和 settlement 屏障必须在同一事务提交，禁止“先改余额后标完成”。恢复差异不通过增发筹码修正；守恒失败则隔离房间并要求人工审计。

## 5. 并发、备份与恢复

应用服务按固定顺序访问 wallet/escrow，乐观版本检查防止陈旧写入；同房命令仍由内存邮箱串行。数据库忙重试只重放具备业务幂等键的整个事务，不能只重放部分 SQL。

WAL 模式备份使用 SQLite Online Backup API 或 `VACUUM INTO` 到临时文件，经 `integrity_check`、大小/校验和验证后原子归档；禁止直接只复制主 `.db` 而遗漏 WAL。备份加密、限制权限、设保留周期且不提交 Git。恢复演练在隔离环境完成：校验 schema 版本、完整性、外键和全局守恒，再切换服务。

## 6. 数据保留与查询投影

会话到期/撤销后按安全策略清理；去重记录按契约过期，结算/流水/审计长期保留。历史 API 从专用读模型白名单映射，不直接返回表行或加密底牌。游客临时数据、练习栈在房间回收后删除或只保留匿名运营计数，不写正式统计。
