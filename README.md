# MetPoker

MetPoker 是面向个人和少量朋友的私人网页端无限注德州扑克项目。项目不涉及真实货币、充值、提现、筹码交易或现金兑换；首要目标是规则正确、牌局公平、资金账目可审计，以及长期可维护。

## 当前状态

当前完成至 **Phase 1A.1：设计契约修正与开发环境检查**。仓库中仍无可运行游戏、数据库或页面；不得把文档中的接口草案和“待实现验收”案例视为已经实现或测试通过。六项产品决策已经项目所有者确认，但在再次完成架构审查前不得进入 Phase 1B。

## 技术栈

- 前端：Vue 3、Vite、Pinia、Vitest
- 后端：PHP 8.2+、Composer、PHPUnit
- 实时服务：Workerman、WebSocket
- 数据存储：SQLite、PDO
- 生产环境：Ubuntu、Nginx、HTTPS/WSS

首版采用单服务器、单权威游戏进程。扑克规则引擎保持纯 PHP、框架无关，不依赖网络、数据库、认证状态或真实系统时钟。

## 目录

```text
backend/                 PHP 后端与测试的预留目录
  src/                   Domain、Application、Infrastructure、Realtime
  tests/                 后端自动化测试
database/migrations/     后续版本化迁移（Phase 1A 不创建数据库）
frontend/src/            Vue 前端的预留目录
docs/                    产品、架构、协议、安全与验收契约
```

## 文档索引

- [产品需求](docs/PRD.md)
- [总体架构](docs/ARCHITECTURE.md)
- [扑克规则](docs/POKER_RULES.md)
- [机器人设计](docs/BOT_DESIGN.md)
- [数据库设计](docs/DATABASE.md)
- [HTTP API 契约](docs/API_CONTRACT.md)
- [WebSocket 协议](docs/WS_PROTOCOL.md)
- [安全设计](docs/SECURITY.md)
- [验收与测试矩阵](docs/ACCEPTANCE.md)
- [开发与部署规划](docs/DEPLOYMENT.md)
- [决策记录](docs/DECISIONS.md)
- [开发路线](docs/ROADMAP.md)

## 开发原则

1. 服务端是牌局与资金的唯一权威；客户端只提交意图并展示按身份裁剪后的投影。
2. Account（登录账号）与 Player（游戏身份）分离；机器人没有登录凭证。
3. 普通房永久筹码与练习房临时筹码完全隔离，所有金额均为非负整数。
4. 管理员无牌局内特权，不得查看未公开牌、操纵发牌、底池或机器人行动。
5. 每一阶段达到验收条件并经确认后，才进入下一阶段。

Phase 1A.1 还固定了短码全押、无人可回应加注、烧牌/发牌、强制改密、游客房主、私有历史牌、行动轮次令牌和重连投影等边界。审查修复矩阵见 [Phase 1A.1 审查记录](docs/reviews/PHASE_1A_1_REVIEW.md)。

本仓库暂不附带开源许可证；许可证选择由项目所有者后续确认。
