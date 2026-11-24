# HTTP API 契约草案

## 1. 通用约定

基址 `/api/v1`，JSON UTF-8，HTTPS only。字段使用 `snake_case`；时间为 UTC ISO-8601；金额为 JSON 整数，禁止浮点和数字字符串。成功响应：`{"data": ..., "meta": {"request_id": "..."}}`；错误响应：`{"error":{"code":"...","message":"中文安全提示","details":{}},"meta":{"request_id":"..."}}`。

修改请求必须带 `X-Request-ID`；去重唯一键为 `(principal_id, request_id)`，资金结算等业务键另行保证全局唯一。影响活跃房间/座位/桌栈的 HTTP 写请求必须转入权威房间命令邮箱，不得直接改表。金额为不超过 `1_000_000_000_000` 的安全整数。

浏览器认证使用 `Secure; HttpOnly; SameSite=Lax` 会话 Cookie；所有非 GET 请求要求 CSRF 双提交令牌/自定义头并校验 Origin。登录、邀请和管理接口限速。WebSocket 通过短时一次性票据连接，不在 URL 长期携带会话令牌。

## 2. 权限符号

- `U`：已认证但必须改密的受限账号，只能读取必要自身认证状态、改密和登出。
- `P`：已登录且未禁用、已完成强制改密的普通/管理员账号。
- `A`：管理员，后端逐接口验证；管理员参局仍只拥有 P 的牌局权限。
- `G`：有效游客 principal；仅能在获授权的练习房使用临时资源。
- `O`：服务端验证的房主 principal，可为注册账号或练习房游客，不等于管理员/旁观者。
- `Self`：只能操作/读取当前主体。

## 3. 身份与会话接口

| 方法与路径 | 权限 | 请求/结果要点 |
|---|---|---|
| `POST /auth/login` | 公开 | username/password；轮换会话，返回自身账号投影与 CSRF token |
| `POST /auth/logout` | U/P | 撤销当前受限或完整会话 |
| `POST /auth/change-password` | U/P | 正常态用 current_password；重置态用一次性 reset credential；成功后轮换为完整会话并使凭证失效 |
| `GET /auth/me` | U/P | 最小账号状态、must_change_password；U 不返回游戏/钱包入口数据 |
| `GET /auth/sessions` | Self | 自己的活跃会话安全元数据 |
| `DELETE /auth/sessions/{id}` | Self | 撤销指定自己的会话 |
| `POST /guest-sessions` | 公开且开关启用 | display_name；创建短时游客会话，不创建 Account/Wallet |
| `POST /registrations` | 公开注册开启或有效邀请码 | 邀请码默认 7 天/一次；消费与 Account + Human Player + 初始钱包同事务，并发只成功一次 |

登录失败统一 `AUTH_INVALID_CREDENTIALS`。管理员重置生成安全随机、仅存摘要、默认 15 分钟有效且一次性的 reset credential，撤销目标全部旧会话/未使用 WS ticket，并终止已认证 WS 连接的后续权限。账号禁用采取同样撤销/断连；当前手按掉线规则安全收尾，不因此作废或没收全押资格。U 状态不能进入游戏、获取 WS ticket、访问钱包或管理接口。

## 4. 房间与牌桌接口

| 方法与路径 | 权限 | 说明 |
|---|---|---|
| `POST /rooms` | P；练习房可 G | G 只能创建练习房并成为 guest owner；普通房 owner 必须为 P |
| `GET /rooms/{id}` | P/G 且有邀请/密码权限 | 返回不含私牌的房间概要 |
| `PATCH /rooms/{id}` | O | 更新配置；响应标注 `effective_immediately` 或 `effective_next_hand` |
| `POST /rooms/{id}/join` | P/G | 加入成员/旁观；普通房拒绝 G |
| `POST /rooms/{id}/seats/{seat}/requests` | P/G 且非旁观授权 | G 仅练习房；请求入座/临时栈，手牌中 queued |
| `DELETE /rooms/{id}/seats/{seat}` | Self 或 O 管理他人 | 请求离桌/踢出；公平相关操作延迟到手末 |
| `POST /rooms/{id}/bots` | O | 添加启用机器人，手牌边界生效 |
| `DELETE /rooms/{id}/bots/{player_id}` | O | 移除机器人，手牌边界生效 |
| `POST /rooms/{id}/buy-ins` | P/Self 或 G/Self | P 普通房永久买入；P/G 练习房仅补临时筹码，永不触达钱包 |
| `POST /rooms/{id}/pause` | 当前唯一在座真人 P/G | 即时冻结当前手；旁观/未入座 owner 无权，房主暂停下一手使用配置命令 |
| `POST /rooms/{id}/resume` | 发起/恢复该单真人暂停的在座 P/G | 恢复当前手；旁观者无权 |
| `POST /rooms/{id}/close` | O/A | 正常关闭排队到当前手安全结算后；仅系统故障恢复能受审计作废，不得用关房撤销亏损 |
| `POST /rooms/{id}/ws-tickets` | P/G 且有房间资源授权 | U/旁观越权请求拒绝；返回 30 秒一次性、绑定 room/principal 的票据 |

实时动作不走 HTTP，统一使用 WS。房间密码只在 join 时验证，存储安全哈希，任何响应不返回密码/哈希。

## 5. 玩家、钱包和历史

| 方法与路径 | 权限 | 说明 |
|---|---|---|
| `GET /players/me` | P | 自己的资料、钱包/托管合计和正式统计 |
| `PATCH /players/me` | P | 仅可改展示昵称等白名单字段并审计；登录用户名和 Player ID 不可改 |
| `GET /players/{id}/public-profile` | P | 仅公开展示字段/统计；机器人公开 difficulty，不返回 personality/策略参数 |
| `GET /wallet/me/transactions` | P | 自己的正式资金流水；不混入练习筹码 |
| `GET /hands` | P | 自己参与的正式手历史，cursor 分页 |
| `GET /hands/{id}` | 注册真人参与者；A 仍按参与关系 | 公共事件、实际公开牌，以及请求者本人加密保存的私牌；A 未参与时无他人私牌 |

历史将 `public_revealed_cards` 与 `my_private_hole_cards` 分离投影。后一字段仅对该有效正式手的注册真人本人解密；机器人未公开牌、作废手私牌和剩余牌堆不存在可查询记录。

## 6. 管理接口

全部要求 A、近期重新认证、CSRF、限速和审计；前端 `/admin` 路由不构成授权。

| 方法与路径 | 说明 |
|---|---|
| `GET/POST /admin/accounts` | 查询/创建账号；创建响应不返回可复用明文密码 |
| `PATCH /admin/accounts/{id}/status` | 禁用/解禁，要求理由；禁用撤销会话 |
| `POST /admin/accounts/{id}/password-reset` | 生成一次性重置流程，撤销全部旧会话，置 must_change_password |
| `GET/POST/PATCH /admin/bots...` | 创建及配置 Bot；性格/难度可管理，修改手末生效；普通接口只公开难度，不公开性格/参数 |
| `POST /admin/wallet-adjustments` | player_id、amount（正/负）、reason；只改钱包、不可改 escrow |
| `GET /admin/ledger-transactions` | 按 player/type/time 查询正式流水 |
| `GET /admin/player-stats` | 真人/机器人正式统计 |
| `GET /admin/rooms` | 活跃房和在线成员；不返回私牌/牌堆 |
| `POST /admin/rooms/{id}/close` | 异常关闭编排，不能直接改池或结算赢家 |
| `GET/PATCH /admin/settings` | 白名单基础设置，敏感密钥不经此接口 |
| `GET /admin/audit-logs` | 只读管理员审计 |

## 7. 关键请求示例

普通房买入：

```json
POST /api/v1/rooms/r_123/buy-ins
{"kind":"permanent","amount":10000,"expected_room_version":7}
```

成功返回钱包与桌托管的新余额、事务 ID 和 queued 状态。相同 X-Request-ID、相同 payload 重试返回相同事务；相同 ID 不同 payload 返回 `409 IDEMPOTENCY_KEY_REUSED`。

管理员扣除：

```json
POST /api/v1/admin/wallet-adjustments
{"player_id":"p_123","amount":-5000,"reason":"纠正经审计确认的错误发放"}
```

若目标可用钱包不足，返回 `409 INSUFFICIENT_WALLET_BALANCE`；不得从其桌上余额扣除。

## 8. 状态码与错误码

| HTTP | 代码示例 | 含义 |
|---:|---|---|
| 400 | `VALIDATION_ERROR` | 格式/范围错误 |
| 401 | `AUTH_REQUIRED`, `AUTH_INVALID_CREDENTIALS` | 未认证/登录失败 |
| 403 | `FORBIDDEN`, `ACCOUNT_DISABLED`, `PASSWORD_CHANGE_REQUIRED` | 身份有效但无权 |
| 404 | `RESOURCE_NOT_FOUND` | 不存在或为防枚举而隐藏 |
| 409 | `VERSION_CONFLICT`, `ROOM_STATE_CONFLICT`, `INSUFFICIENT_WALLET_BALANCE`, `IDEMPOTENCY_KEY_REUSED` | 状态冲突 |
| 422 | `RULE_VIOLATION`, `INVALID_BUY_IN` | 语义合法但违反领域规则 |
| 429 | `RATE_LIMITED` | 限速，带 Retry-After |
| 503 | `DATABASE_BUSY`, `SERVICE_RECOVERING` | 有界重试后不可用 |

错误 details 只含安全字段（如当前版本、合法范围）；非法动作不得泄露牌堆或他人状态。
