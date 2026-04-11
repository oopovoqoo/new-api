# new-api 项目记忆

## 项目概况
- AI API 网关/代理，Go 1.22+ + Gin + GORM，支持 40+ 上游 AI 提供商
- 前端：React 18 + Vite + Semi Design，包管理器用 bun
- 数据库：SQLite / MySQL / PostgreSQL 三库兼容
- 缓存：Redis + 内存缓存

## 商业化方向
- 面向中国大陆 ToC 用户运营
- 遵守 AGPLv3：代码改动开源，配置文件（密钥、Logo、收款码）不入仓库
- 需开发：手机号注册（SMS）、微信支付/支付宝、实名制、内容安全对接、发票系统

## 文档目录（docs/）
- `docs/security-audit.md` — 安全审计记录（2026-03-09，基于 v0.11，修复前需重新确认行号）
- `docs/DEV_GUIDE.md` — 开发指南，含本地启动、环境变量、手动出包方法
- `docs/中国大陆商业化改造方案.md` — 合规与功能改造方案（2026-03，基于 v0.11/v0.12）
- `docs/self-upgrade-feature.md` — 在线自升级功能改造方案（待实现）
