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

`request_id` 对命令必填且在主体范围唯一；`hand_id` 对手牌命令必填；`expected_version` 防止陈旧动作。服务端按房间串行验证。相同 request_id+payload 返回原结果，不再次执行；相同 ID 不同 payload 关闭/告警。版本不匹配返回 `STALE_STATE` 和安全的 current_version，并要求同步。

## 3. 客户端命令

| type | payload | 规则 |
|---|---|---|
| `session.authenticate` | ticket, last_seen_version? | 连接首条；成功后快照或连续补发 |
| `room.sync.request` | last_seen_version | 主动请求补发/快照 |
| `room.ready` | ready boolean | 两手之间设置准备状态 |
| `game.action` | action, target_round_total? | 仅当前行动者；目标为本轮累计额 |
| `room.pause.request` | reason | 仅单真人规则允许 |
| `room.resume.request` | 无 | 恢复冻结计时 |
| `room.leave.request` | 无 | 手中则排队到安全边界 |
| `connection.pong` | ping_id | 心跳响应 |

入座、买入、管理、配置用 HTTP 获得持久幂等事务；WS 负责低延迟牌局动作。服务端可把 HTTP 造成的变化作为 WS 事件广播。

## 4. 快照与增量事件

认证/重连后，若 `last_seen_version` 到当前版本的事件仍连续缓存，按序补发；否则发送 `room.snapshot`。快照包含 `version`、房间公开配置、生效待办、座位公开状态、当前玩家自己的可见状态、公共牌、各池公开金额、行动者、服务器截止时间/暂停剩余时间和自身合法动作。

主要事件：

- `room.snapshot`、`room.member.joined/left`、`room.owner.changed`、`room.config.pending/applied`
- `seat.occupied/vacated`、`buy_in.applied`（金额仅按权限投影）
- `hand.started`、`blind.posted`、`turn.started`、`game.action.accepted`
- `board.dealt`、`player.cards.dealt`（只单播给本人）、`player.cards.revealed`（仅实际公开）
- `pot.updated`、`hand.showdown`、`pot.awarded`、`hand.settled`、`hand.voided`
- `room.paused/resumed/closed`、`player.disconnected/reconnected`
- `command.rejected`、`sync.required`、`connection.ping`

增量事件版本严格递增；一个命令产生的一组事件可共享最终 version 和 batch_id，客户端原子应用。检测到缺号、逆序或未知关键事件时，停止本地动作并请求快照。客户端 Pinia 不从事件自行推导隐藏状态或裁定赢家。

## 5. 身份投影

同一内部事件按接收者生成不同 payload：

- 当前玩家收到自己的 `hole_cards`；其他玩家和旁观者不含该字段。
- 摊牌只广播规则上实际公开的牌；muck 牌永不在事后快照/历史出现。
- 机器人策略不订阅网络广播，而由服务端白名单构造 context。
- 管理员旁观与普通旁观完全相同，不获得额外字段。
- 连接日志、拒绝消息不回显原始 ticket、Cookie 或私牌。

严禁广播内部 `deck`、烧牌、随机种子、所有玩家底牌、密码哈希、账号角色细节或完整数据库对象。

## 6. 行动与倒计时

`turn.started` 提供 `acting_player_id`、`deadline_server_time`、`duration_ms`，只有行动者投影额外包含 `legal_actions`：每类动作、call_amount、min/max target 和 raise_reopened。客户端倒计时仅展示；服务端时钟决定是否超时。

暂停事件包含冻结的 `remaining_ms`，不含新的 deadline。恢复生成新 deadline。机器人延迟和超时都提交带原版本的内部命令；若玩家先行动或状态变化，旧命令被拒绝且不改变状态。

## 7. 拒绝、重试与关闭码

`command.rejected.payload` 包含 `code`（`NOT_YOUR_TURN`, `ILLEGAL_ACTION`, `STALE_STATE`, `HAND_MISMATCH`, `ROOM_PAUSED`, `RATE_LIMITED`）、安全说明和 current_version。非法动作不推进版本。网络未知结果时客户端用相同 request_id 重试，不能生成新 ID 造成双动作。

应用关闭码建议：4001 未认证、4003 无权限/账号禁用、4008 限速、4009 版本需重同步、4010 房间关闭、1011 服务端异常。可恢复断开使用指数退避加抖动；`SERVICE_RECOVERING` 时停止自动高频重连。

## 8. 重连案例

玩家在 version 40 掉线，房间推进至 44：若缓存仍有 41–44，认证后补发并最终发送 `sync.complete(44)`；若缓存从 43 才开始，则发送 version 44 的裁剪快照。若该玩家是单真人，掉线先产生 paused 事件并冻结计时；多人桌继续，超时可能自动 check/fold。已全押玩家重连与否不改变领奖资格。
