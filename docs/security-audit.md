# 安全审计记录（2026-03-09）

> **版本说明**：本次审计基于 v0.11 代码截面。升级到 v0.12.x 后，以下行号可能已发生变化，
> 实施修复前请先对照当前代码重新确认问题是否仍然存在。

详细修复方案见 `/Users/reputationly/.claude/plans/immutable-pondering-book.md`

## P0 — 上线前必须修复

1. **Cookie Secure=false** (`main.go:177`) — 改为 true 或环境变量控制
2. **密码重置返回明文密码** (`controller/misc.go:367-371`) — 改为仅邮件发送，响应不含密码
3. **Password 字段 json tag 未隐藏** (`model/user.go:26`) — 改为 `json:"-"`

## P1 — 本周修复

4. **邀请码仅 4 字符** (`controller/user.go:351`) — 改为 12 位 + 接口限流
5. **Token 额度 TOCTOU 竞态** (`service/quota.go:459-462`, `model/token.go:393-421`) — 用 DB 原子 SQL 替代 Go 层预检查
6. **Creem webhook 测试模式绕过签名** (`controller/topup_creem.go:38-50`) — 生产模式禁用测试模式

## P2 — 下次迭代

7. **邮箱枚举** (`controller/misc.go:311-315`) — 统一返回模糊提示
8. **BatchUpdate 加剧竞态** (`model/token.go:405-408`) — Redis 原子扣减

## P3 — 技术债

9. **auth 中间件类型断言无 recover** (`middleware/auth.go:104,112,120`)
10. **UpdateSelf 反序列化到完整 User struct** (`controller/user.go:688-694`)
11. **Channel test 日志含 API Key** (`controller/channel-test.go:270,499`)
