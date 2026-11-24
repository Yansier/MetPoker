# 开发与部署规划

本文是规划而非已执行部署。Phase 1A 不安装依赖、不启动服务、不连接生产环境。

## 1. Windows 开发环境

建议版本：Git、PHP 8.2+（扩展 PDO SQLite、mbstring、openssl、sodium）、Composer 2、Node.js 当前 LTS、npm。开发者复制 `.env.example` 为未跟踪的 `.env`，使用独立本地 SQLite 文件；不得把绝对个人路径写入提交配置。

规划命令（相应阶段加入配置后才可用）：

```powershell
composer install
npm ci --prefix frontend
composer test
npm test --prefix frontend
```

Workerman 生产目标为 Linux；Windows 仅作开发/自动化测试。时间统一 UTC 存储，UI 转用户时区。随机/时钟测试通过注入，不改系统时钟。

## 2. Ubuntu 生产拓扑

```text
Internet -> Nginx :443 -> PHP HTTP service
                      └-> Workerman WSS service (loopback)
Application user -> SQLite/WAL + logs + encrypted backups
```

Nginx 终止 TLS，HTTP 与 WS 反代到 loopback，不直接暴露应用端口。静态前端由 Nginx 提供。PHP HTTP 形态（PHP-FPM 或同项目明确入口）在 Phase 2A 决定；Workerman 由 systemd 管理、单权威 game worker 起步。服务使用非 root 专用用户，数据库/密钥目录最小权限且不在 Web 根目录。

## 3. 配置与秘密

环境至少包含：APP_ENV、公开基址、允许 Origin、数据库路径、session/加密密钥引用、日志级别、注册/游客开关、初始筹码、掉线/回收期限和备份目标。仓库只提交无秘密的 `.env.example`；生产秘密由权限受控环境文件或 secret 管理器注入，禁止写日志或管理设置表。

生产启动必须断言：APP_ENV=production、HTTPS/WSS 基址、安全随机适配器、无调试牌堆/固定 seed、数据库 schema 版本匹配、目录权限正确。任一失败拒绝启动游戏流量。

## 4. 部署步骤（规划）

1. 在预发布环境从锁文件构建前后端，运行测试、依赖审计和 secret scan，生成不可变发布包及校验和。
2. 备份并验证数据库；进入维护/禁止开新手，等待进行中手安全结束。
3. 停止应用，执行经审查的迁移；迁移失败按预案恢复备份，不能继续带错版本启动。
4. 原子切换 release 软链接，启动 HTTP，再运行崩溃恢复/守恒检查，最后启动 Workerman 接流量。
5. 执行登录、练习房、普通房只读/小额受控冒烟和 WS 重连检查；观察错误、busy 和守恒指标。
6. 失败则停止新流量，按数据库兼容性选择代码回滚或数据库恢复；所有操作记运维记录。

## 5. Nginx 与 systemd 要点

Nginx 配置 TLS 1.2+、HSTS（确认全站 HTTPS 后）、CSP、nosniff、frame-ancestors、请求体/速率限制；WS 转发 Upgrade/Connection，设置合理读超时但依赖应用心跳发现死连接。敏感路径、`.env`、数据库和备份永不可由静态服务器访问。

systemd 设置 `Restart=on-failure`、受限文件系统/能力、明确工作目录和环境文件权限；停止时先下线健康检查、拒绝新手、给予当前命令短暂完成窗口。强杀后的恢复遵循 `DATABASE.md`，不尝试从不可信内存续局。

## 6. 健康、日志与监控

存活检查只判断进程；就绪检查验证 schema、恢复任务完成、数据库可读写和权威 worker 可用，但不返回秘密。指标/告警：登录失败峰值、WS 连接/重连、房间命令延迟、事件版本错误、SQLite busy、结算/作废/恢复、备份失败、磁盘空间、守恒差异。

结构化日志脱敏并轮转，字段含 request/room/hand/version，不含密码、Cookie/ticket、私牌、牌堆或真实数据库行。服务器时间通过受控 NTP 同步；牌局倒计时仍用单调时钟适配器。

## 7. 备份、恢复与维护

按实际游戏频率制定至少每日加密在线备份和更短 WAL 检查点策略；保留多代且存放在与主机故障域不同的位置。每月至少在隔离环境恢复一次，运行 `integrity_check`、外键检查、schema 校验和全局筹码守恒。

升级/关机先禁止新手并等待手末；无法等待时停止后由启动恢复作废 STARTED 手。旧房间在恢复后关闭并幂等返还 escrow。数据库手工编辑禁止作为常规修复手段。

## 8. 上线门槛

只有 Phase 4A 无阻断缺陷、HTTPS/WSS 和权限审计通过、备份恢复成功、异常重启三窗口演练通过、真实多人体验验收及回滚预案确认后，才能进入 Phase 4B。上线不改变“私人、无真实货币”的产品边界。
