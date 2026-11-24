# Phase 1A.1 设计契约审查记录

日期：2026-11-24。基准：Phase 1A 提交 `4e4188c`。本轮只修订文档、仓库卫生和环境结论，不包含业务代码、数据库、依赖安装或可运行测试。

## 1. 六项所有者决策

| 编号 | 已确认结果 | 主要落点 |
|---|---|---|
| Q-001 | 机器人难度公开、性格及策略参数隐藏，配置手末生效 | PRD、BOT_DESIGN、API、WS、SECURITY、DATABASE、DECISIONS |
| Q-002 | 零桌栈才手末再买；默认额优先，余额达到最低则用剩余，低于最低离桌；正栈不补满 | PRD、BOT_DESIGN、ACCEPTANCE、DECISIONS |
| Q-003 | 首版无自愿展示弃牌或 SHOW_CARDS | POKER_RULES、DECISIONS |
| Q-004 | 邀请码默认 7 天/一次，安全摘要，同事务消费并防并发复用 | PRD、DATABASE、API、DECISIONS |
| Q-005 | 用户名不可改，昵称可安全修改并审计，稳定 ID 归属 | PRD、API、SECURITY、DECISIONS |
| Q-006 | 暂不添加 LICENSE | README、DECISIONS |

## 2. P1 阻断问题

| 编号 | 原问题 | 修订与验收结论 |
|---|---|---|
| P1-01 | 累计短全押案例缺少可回应者，混淆重开权与可下注性 | 增加四人可回应、三人无人回应及逐玩家权利三案；明确不能用全局布尔值。文档通过，代码测试待 Phase 1B。 |
| P1-02 | 弃牌边池案例无法由合法序列产生 | 改为 C300/A600/B1000/D1000/A弃的四人序列，池为1200/900/800；保留三人未跟注退还案。代码测试待实现。 |
| P1-03 | all_in 可能被解释为绕过加注权 | 按目标额分类；无加注权只能全押跟注，余栈大于 call 时不能 all-in，提高最高额需合法加注权及可回应对手。 |
| P1-04 | 发底牌起点与烧牌不够确定 | 固定按钮左侧起、逐张两轮；每条公共街先烧牌；单挑大盲先收牌；提前结束不跑牌。确定性测试待实现。 |

## 3. P2 跨文档问题

| 编号 | 原问题 | 修订与验收结论 |
|---|---|---|
| P2-01 | 强制改密接口被完整权限挡住 | 新增 U 受限认证态；15 分钟一次性重置凭证；可查认证状态/改密/登出，禁止游戏/钱包/管理；撤销会话/ticket/连接权限。 |
| P2-02 | 历史私牌保留与“不保留未公开牌”冲突 | 公开牌与本人私牌分栏；有效正式手只加密保留注册真人本人牌；管理员无越权，机器人/作废手不留；明确 root 信任边界。 |
| P2-03 | 净盈亏可能把未跟注退还算作奖励 | 固定 gross、uncalled、net committed、pot awarded 四字段及两条等价公式；全手净和为零。 |
| P2-04 | rooms 只支持 owner_account_id，无法表示游客房主 | 改为 account/guest 两个可空 FK 且恰有一个；普通房只 account，机器人禁为 owner。 |
| P2-05 | 游客可参加练习房但 API/WS 权限不完整 | 明确创建、管理、入座、补临时筹码、动作、单真人暂停/恢复、离桌；旁观无动作权，永不触达正式资金路径。 |
| P2-06 | state_version 无关变化会误杀计时器 | 引入 hand_id、turn_id、acting_player_id、timer_generation；版本继续用于同步，任务按行动身份判定。 |
| P2-07 | 单真人自动暂停可能无限锁资 | 保留120秒；超期退出暂停、自动check/fold收尾、手末返还且不再开手；离桌/关房不能撤销亏损。 |
| P2-08 | 直接补发内部事件可能泄露私有内容 | 重认证总先裁剪快照；同连接缺口才逐批重投影；纯他人私有批次用空确认推进；不安全则快照。 |

## 4. 补充架构澄清

- 同一永久 Player 同时最多一个有效座位；机器人相同，旁观不受限。
- HTTP/WS 房间写入统一进入权威邮箱，独立进程不能直接改活跃座位/桌栈。
- 进行中 escrow 保持开手检查点口径，实时栈/投入/池在内存拆分，结算事务才更新最终 escrow。
- escrow 分录使用 room_id + player_id；请求去重使用 principal_id + request_id，业务键全局唯一。
- 流水/净盈亏可负，余额/栈/承诺/池不可负；所有金额及中间结果上限为 1,000,000,000,000。

## 5. 环境检查结论

Git 2.49.0 可用。当前命令、常见安装目录、已安装程序登记和 Codex 运行时均未发现 PHP 或 Composer；因此无法验证 PHP 位数、版本、ini、Composer 版本，以及 PDO、pdo_sqlite、sqlite3、mbstring、openssl、sodium 扩展。用户 Node/npm shim 未激活版本，SQLite CLI 未发现；二者不阻断 Phase 1B。

截至本记录日期，建议后续由项目所有者从 [PHP 官方 Windows 下载页](https://www.php.net/downloads.php?os=windows)选择 PHP 8.5 x64 NTS（兼顾稳定支持期与 CLI 用途），再从 [Composer 官方下载页](https://getcomposer.org/download/)安装 Composer 2；[PHPUnit 支持矩阵](https://phpunit.de/supported-versions.html)显示 PHPUnit 13 需要 PHP 8.4+。实际选择仍须在进入 Phase 1B 前确认并验证扩展。

结论：设计契约具备 Phase 1A.1 复审条件；本机 Phase 1B 环境尚未就绪，至少缺少可验证的 64 位 PHP 8.2+ 与 Composer 2。所有规则与协议案例均为待实现验收，不声称测试已通过。
