# WebSocket 协议契约

## 1. 连接与信任边界

客户端先通过 HTTP 获取 30 秒有效的一次性 room-bound ticket，再连接 WSS。首条消息 `session.authenticate` 提交票据；服务端绑定 principal、player/guest、room 和连接 ID。票据使用一次即失效，重连获取新票据。生产拒绝明文 WS、跨 Origin 和客户端自报角色。

服务端心跳：每 20 秒 ping，客户端 10 秒内 pong；连续失败关闭连接并触发断线流程。限制消息大小、频率和 JSON 深度；未知类型/字段按协议错误处理，不反序列化任意对象。

## 2. 统一信封

客户端命令：

```json
{
  "type": "game.action",
  "request_id": "019...",
  "room_id": "r_123",
  "hand_id": "h_456",
  "turn_id": "t_009",
  "expected_version": 42,
  "sent_at": "2026-01-01T00:00:00.000Z",
  "payload": {"action":"raise","target_round_total":1200}
}
```

服务端消息：

```json
{
  "type": "game.action.accepted",
  "event_id": "e_789",
  "request_id": "019...",
  "room_id": "r_123",
  "hand_id": "h_456",
  "version": 43,
  "server_time": "2026-01-01T00:00:00.100Z",
  "payload": {}
}
```

`request_id` 对命令必填，去重键为 `(principal_id, request_id)`；手牌动作还必须带 `hand_id` 与服务端下发的 `turn_id`。`expected_version` 用于普通陈旧状态校验；对 `game.action`，只要 turn 身份仍匹配且动作在当前合法集合，纯粹由旁观者加入等无关事件造成的版本差异不能拒绝有效动作。超时/机器人任务同样按 turn 身份判断。

## 3. 客户端命令

| type | payload | 规则 |
|---|---|---|
| `session.authenticate` | ticket | 连接首条；成功后总是先发送当前主体裁剪快照 |
| `room.sync.request` | last_seen_version | 已认证连接请求连续安全批次；不可安全投影则快照 |
| `room.ready` | ready boolean | 两手之间设置准备状态 |
| `game.action` | hand_id, turn_id, action, target_round_total? | 仅当前行动者；all_in 目标由服务端计算且不能绕过加注权 |
| `room.pause.request` | reason | 仅单真人规则允许 |
| `room.resume.request` | 无 | 恢复冻结计时 |
| `room.leave.request` | 无 | 手中则排队到安全边界 |
| `connection.pong` | ping_id | 心跳响应 |

入座、买入、管理、配置用 HTTP 获得持久幂等事务；WS 负责低延迟牌局动作。服务端可把 HTTP 造成的变化作为 WS 事件广播。

游客在授权练习房中可提交游戏动作、离桌，并在自己是唯一在座真人时暂停/恢复；guest owner 可管理其练习房。普通房拒绝游客。旁观者即使是 room member 也无牌局动作、暂停或恢复权。所有这些写操作与 HTTP 写操作进入同一权威房间邮箱。

## 4. 快照与增量事件

新连接和重新认证后总是发送 `room.snapshot`，作为唯一同步起点。快照包含 version、房间配置、座位公开状态、当前主体可见状态、公共牌、池、`hand_id/turn_id/acting_player_id/timer_generation`、截止时间及自身合法动作。只有同一已认证连接出现短暂缺口时，服务端才考虑补发连续事件批次。

主要事件：

- `room.snapshot`、`room.member.joined/left`、`room.owner.changed`、`room.config.pending/applied`
- `seat.occupied/vacated`、`buy_in.applied`（金额仅按权限投影）
- `hand.started`、`blind.posted`、`turn.started`、`game.action.accepted`
- `board.dealt`、`player.cards.dealt`（只单播给本人）、`player.cards.revealed`（仅实际公开）
- `pot.updated`、`hand.showdown`、`pot.awarded`、`hand.settled`、`hand.voided`
- `room.paused/resumed/closed`、`player.disconnected/reconnected`
- `command.rejected`、`sync.required`、`connection.ping`

增量按 batch 严格递增版本，一个命令的事件共享 batch_id/version 并原子应用。每次补发都重新验证当前认证主体、房间成员/参与关系和私有事件所属玩家，再从内部事件重建接收者投影，禁止复用他人的缓存响应。若某版本只含其他玩家私有事件，发送不含私有 payload 的 `projection.advanced` 空批次确认版本；无法安全判定历史可见性时发送新快照。

## 5. 身份投影

同一内部事件按接收者生成不同 payload：

- 当前玩家实时收到自己的 `hole_cards`；其他玩家和旁观者不含该字段。机器人难度可公开，性格与策略参数永不进入普通/旁观投影。
- 摊牌只广播规则上实际公开的牌；muck 牌永不进入实时、重连或公共历史投影。注册真人本人的加密历史牌只经 HTTP 本人接口提供，不通过 WS 补发。
- 机器人策略不订阅网络广播，而由服务端白名单构造 context。
- 管理员旁观与普通旁观完全相同，不获得额外字段。
- 连接日志、拒绝消息不回显原始 ticket、Cookie 或私牌。

严禁广播内部 `deck`、烧牌、随机种子、所有玩家底牌、密码哈希、账号角色细节或完整数据库对象。

## 6. 行动与倒计时

`turn.started` 提供 hand_id、turn_id、acting_player_id、timer_generation、deadline 和 duration；行动者额外收到来自同一领域规则源的 legal_actions。旁观者加入不改变这些字段或延长期限。客户端倒计时仅展示；服务端时钟决定超时。

暂停事件包含冻结的 remaining_ms 并使旧 timer_generation 失效；恢复创建新 generation/deadline。玩家行动、暂停/恢复、行动者切换和手牌结束均撤销旧任务。机器人延迟及超时提交 turn 身份；无关 state_version 改变不重启思考，旧 turn 任务无害拒绝。

## 7. 拒绝、重试与关闭码

`command.rejected.payload` 包含 `code`（`NOT_YOUR_TURN`, `ILLEGAL_ACTION`, `STALE_STATE`, `HAND_MISMATCH`, `ROOM_PAUSED`, `RATE_LIMITED`）、安全说明和 current_version。非法动作不推进版本。网络未知结果时客户端用相同 request_id 重试，不能生成新 ID 造成双动作。

应用关闭码建议：4001 未认证、4003 无权限/账号禁用、4008 限速、4009 版本需重同步、4010 房间关闭、1011 服务端异常。可恢复断开使用指数退避加抖动；`SERVICE_RECOVERING` 时停止自动高频重连。

## 8. 重连案例

玩家在 version 40 掉线、房间推进至 44：重新认证后直接取得 version 44 的身份裁剪快照，不回放未经重新授权的旧私有事件。若同一认证连接只漏了 42–44，则可按上述批次规则补发。单真人意外断线先自动暂停，120 秒保留期届满仍未重连则退出暂停、自动 check/fold 收尾；已全押资格不受影响。
