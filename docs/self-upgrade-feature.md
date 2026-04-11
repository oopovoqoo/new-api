# 在线自升级功能改造方案

参考 sub2api 项目的实现，为 new-api 增加真正可执行的在线升级能力（当前仅为通知展示）。

## 原理

1. 后端下载 GitHub Releases 中对应平台的新版二进制文件
2. 校验 SHA256 checksum
3. 原子替换当前可执行文件（`os.Rename`）
4. 调用 `os.Exit(0)`，Docker 的 `restart: always` 策略自动重启容器
5. 前端轮询 `/api/status` 直到服务恢复，自动刷新页面

> **前提**：docker-compose 中 new-api 服务必须配置 `restart: always`

---

## 需要新建的文件

### `service/update.go`

核心升级逻辑：

- `FetchLatestRelease(ctx)` — 调 GitHub API 获取最新 release，按平台匹配资产文件
  - linux/amd64 → `new-api-v*`
  - linux/arm64 → `new-api-arm64-v*`
  - darwin → `new-api-macos-v*`
  - windows → `new-api-*.exe`
  - 同时查找对应的 `checksums-{linux,macos,windows}.txt`

- `PerformUpdate(ctx)` — 执行升级
  1. 版本号为 `v0.0.0` 时拒绝（源码构建不支持自升级）
  2. 加内存互斥锁，防并发
  3. 调 `FetchLatestRelease` 拿下载地址
  4. 已是最新版则返回 `needRestart=false`
  5. 下载到与当前可执行文件同一目录的临时文件（保证 `os.Rename` 原子性）
  6. 校验 SHA256
  7. `os.Rename(exePath, exePath+".backup")`
  8. `os.Rename(tmpPath, exePath)`
  9. 失败时从 backup 回滚

- `RollbackUpdate()` — 将 `.backup` 文件还原

安全控制：仅允许 `github.com` / `objects.githubusercontent.com` 域名、强制 HTTPS、最大 200MB、必须通过 checksum 校验、防路径穿越。

---

### `common/restart.go`

```go
func ScheduleRestart() {
    go func() {
        time.Sleep(500 * time.Millisecond)
        os.Exit(0)
    }()
}
```

---

### `controller/system.go`

三个 handler，遵循现有 `gin.H{"success": bool, "message": str, "data": ...}` 规范：

| Handler | 说明 |
|---------|------|
| `CheckUpdate` | 调 `FetchLatestRelease`，返回版本对比信息和 changelog |
| `PerformUpdate` | 调 `PerformUpdate`，返回 `need_restart: true` |
| `RestartService` | 调 `ScheduleRestart()`，立即返回响应后进程退出 |

---

## 需要修改的文件

### `router/api-router.go`

在 `RootAuth()` 分组下新增（与 `/option` 路由同级）：

```go
systemRoute := apiRouter.Group("/system")
systemRoute.Use(middleware.RootAuth())
{
    systemRoute.GET("/check-update",  controller.CheckUpdate)
    systemRoute.POST("/update",       controller.PerformUpdate)
    systemRoute.POST("/restart",      controller.RestartService)
}
```

### `web/src/components/settings/OtherSetting.jsx`

在现有系统信息卡片内扩展（`checkUpdate` 函数约 231 行起），原有弹窗逻辑保持不变：

**新增状态：**
```js
const [updating, setUpdating]     = useState(false)
const [restarting, setRestarting] = useState(false)
const [needRestart, setNeedRestart] = useState(false)
```

**新增函数：**
- `handleUpdate()` — POST `/api/system/update`，成功后 `needRestart=true`
- `handleRestart()` — POST `/api/system/restart`，触发轮询
- `pollUntilAlive()` — 每 2 秒 GET `/api/status`，最多 30 次，成功后 `window.location.reload()`

**UI 状态流：**
```
检查更新 → 发现新版本 → [立即安装] 按钮
    ↓ 点击
  [正在下载...] spinner
    ↓ 完成
  [立即重启] 按钮
    ↓ 点击
  [正在重启... 倒计时] spinner
    ↓ 服务恢复
  自动刷新页面
```

---

## 实现注意事项

1. **不要复用 `service.DoDownloadRequest`** — 该函数有 SSRF 防护，会拦截 GitHub 域名，update.go 需要自己实现 HTTP 客户端
2. **JSON 操作用 `common.Marshal` / `common.Unmarshal`** — 不直接用 `encoding/json`
3. **版本比较**：使用语义化版本比较（参考 sub2api 的 `compareVersions` 实现）
4. **Windows 支持**：Windows 无法直接替换运行中的 `.exe`，需要用重命名技巧或提示用户手动升级

---

## 参考实现

sub2api 对应文件：
- `backend/internal/service/update_service.go` — 下载、校验、替换逻辑
- `backend/internal/handler/admin/system_handler.go` — handler 层
- `backend/internal/pkg/sysutil/restart.go` — 重启逻辑
- `frontend/src/components/common/VersionBadge.vue` — 前端升级交互
