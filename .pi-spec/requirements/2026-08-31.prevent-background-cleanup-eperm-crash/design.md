# 防止后台清理权限错误导致 pi 退出：设计

## 1. 现状

- `extensions/background-bash/src/process-manager.ts` 的 `ProcessManager` 负责创建独立进程组、等待命令退出、清理进程组以及维护前台/后台命令状态。
- 顶层 `killProcessGroup(pgid)` 调用 `process.kill(-pgid, "SIGKILL")`；它仅静默处理表示进程组已不存在的 `ESRCH`，其他错误都会继续抛出。
- `ProcessManager.spawnProcess()` 通过 `child.once("exit", () => killProcessGroup(pgid))` 在命令进程自然退出时自动清理同组遗留进程。该事件回调没有错误边界，因而 `EPERM` 会作为未捕获异常离开事件回调并终止宿主 pi 进程。
- 同一个 `killProcessGroup()` 还用于用户主动终止、超时、会话关闭、日志失败及宿主退出等路径；这些路径不属于本需求，不能通过全局放宽错误处理改变其现有结果。
- `extensions/background-bash/tests/background-bash.integration.test.ts` 使用真实扩展注册流程构建会话级测试夹具，能够在同一会话连续执行命令、观察会话消息，并对 `process.kill` 做局部替身。现有测试已覆盖自然退出清理、主动终止、超时和会话关闭。
- 该扩展使用 Vitest，包内验证入口为 `vitest --run` 与 `tsc -p tsconfig.json --noEmit`。`@richardgill/pi-background-bash` 是公开包，仓库约定可发布修复附带 Changesets 补丁说明。

## 2. 改动点

### R-1 自动清理权限错误隔离

1. 在 `extensions/background-bash/src/process-manager.ts` 新增仅服务于“命令退出后的自动进程组清理”的错误边界：
   - 调用现有 `killProcessGroup(pgid)`；
   - 仅当抛出错误的 `code` 为 `EPERM` 时静默结束；
   - 其他错误继续抛出；
   - 不重试，不产生消息。
2. 将 `child` 的 `exit` 监听器改为调用上述专用自动清理函数。主动终止、超时、会话关闭及其他调用点继续直接调用 `killProcessGroup()`，保持原行为。
3. 在 `extensions/background-bash/tests/background-bash.integration.test.ts` 增加 AC-1 回归测试：仅让负进程组 `SIGKILL` 返回 `EPERM`，执行自然退出命令后在同一测试会话再次执行命令，并断言第二次执行成功且没有新增会话消息。测试结束时恢复 `process.kill`，避免替身泄漏到其他用例。
4. 在 `.changeset/quiet-background-cleanup.md` 为 `@richardgill/pi-background-bash` 添加 patch 级发布说明，说明自动清理遇到权限拒绝时不再终止会话。

## 3. 文件清单

- 修改：`/Users/bb/Projects/pi-extensions/extensions/background-bash/src/process-manager.ts`
- 修改：`/Users/bb/Projects/pi-extensions/extensions/background-bash/tests/background-bash.integration.test.ts`
- 新增：`/Users/bb/Projects/pi-extensions/.changeset/quiet-background-cleanup.md`

## 4. 数据与接口变更

- 数据结构、持久化格式、命令行接口、工具输入输出、配置项：无变更。
- 新增内部自动清理边界的完整逻辑如下；现有 `killProcessGroup()` 本身保持不变：

```ts
const cleanupExitedProcessGroup = (pgid: number): void => {
  try {
    killProcessGroup(pgid);
  } catch (error) {
    const code = (error as NodeJS.ErrnoException).code;
    if (code !== "EPERM") throw error;
  }
};
```

- 自然退出监听器的完整调用形式改为：

```ts
child.once("exit", () => cleanupExitedProcessGroup(pgid));
```

- Changeset 完整内容为：

```md
---
"@richardgill/pi-background-bash": patch
---

Prevent an automatic background process-group cleanup permission error from terminating the Pi session.
```

## 5. 测试策略

- 测试框架：Vitest，Node 环境。
- AC-1 使用 `extensions/background-bash/tests/background-bash.integration.test.ts` 的会话级集成夹具：
  1. 保存真实 `process.kill` 并安装局部 spy；仅对负 PID 且信号为 `SIGKILL` 的调用抛出带 `code: "EPERM"` 的错误，其余调用转发给真实实现。
  2. 在夹具中执行一个会自然退出的前台命令，触发 `exit` 自动清理。
  3. 在同一夹具中执行第二个命令，断言结果正常返回，以证明宿主会话仍可接收操作。
  4. 断言 `messages` 没有新增自动清理错误消息。
  5. 在 `finally` 中恢复 spy，并正常关闭夹具。
- 运行整个 background-bash 集成测试文件，以同时回归主动终止、超时、自然退出遗留进程清理和会话关闭行为。
- 运行包内 TypeScript 检查，确认错误收窄与 `process.kill` 替身符合现有类型约束。
- 最小验证命令：
  `pnpm --dir /Users/bb/Projects/pi-extensions --filter @richardgill/pi-background-bash exec vitest --run tests/background-bash.integration.test.ts && pnpm --dir /Users/bb/Projects/pi-extensions --filter @richardgill/pi-background-bash tsc`

## 6. 备选方案与取舍

### 方案 A：只在自然退出自动清理边界忽略 EPERM

新增专用自动清理函数，并仅替换 `child` 的 `exit` 监听器调用。该方案把错误隔离限制在 R-1 指定的触发路径，其他 `killProcessGroup()` 调用方保持原语义。

### 方案 B：让 killProcessGroup 全局忽略 EPERM

在现有 `killProcessGroup()` 中把 `EPERM` 与 `ESRCH` 一并静默处理。代码行数略少，但会同时吞掉用户主动终止、超时、会话关闭和日志失败路径中的权限错误，违反明确的非目标，因此否定。

### 根源性自检

1. **触及根源还是缓解症状？** 根源是同步 `exit` 事件回调缺少针对自动清理权限失败的边界；方案 A 直接在该边界收窄错误，而不是通过全局未捕获异常处理器掩盖崩溃。
2. **有无更优做法？** 将监听器改成异步函数或增加全局异常监听都不能提供更窄、更可靠的错误归属；全局修改通用清理函数又会扩大行为变化。无需新增抽象层或第三方依赖。

**推荐：方案 A。** 它以最小改动精确满足 R-1，并保持所有非目标路径的现有行为。
