# 修复说明：`Add KernelSU Next` 以 exit 127 失败

## 现象

- 运行：<https://github.com/gxy1110/Action-Build/actions/runs/39086034170/job/117552689926>（分支 `KernelSU-Next`，提交 `6424700`）
- 失败步骤：第 25 步 `Add KernelSU Next`，耗时约 **215 ms**，退出码 **127**
- 其后 `Apply Fake Patches` … `Build Kernel FAST` 等 30 多个步骤全部 `skipped`
  —— **根本没有进入编译阶段**，所以这不是编译错误

日志关键行：

```text
KSU_META="main/builtin/Numbersf/"
手动指定 hash: Numbersf
bash: line 1: 404:: command not found
##[error]Process completed with exit code 127.
```

## 根因

1. 这次 dispatch 填的 `KSU_META` 还是旧 SukiSU 格式的值 `main/builtin/Numbersf/`
   （新工作流的默认值是 `dev/dev-susfs/`）。
2. 脚本把第一段当作“管理器分支”，拼出
   `https://raw.githubusercontent.com/pershoot/KernelSU-Next/refs/heads/main/kernel/setup.sh`。
3. `pershoot/KernelSU-Next` 没有 `main` 分支（默认分支是 `dev`），该地址返回 **404**。
4. 致命点：`curl -LSs` 少了 `-f`。404 时 curl 仍然退出 0，把响应正文 `404: Not Found`
   原样管道给 `bash -s`，bash 把 `404:` 当命令执行 → `command not found` → **exit 127**。

## 本次改动（只改 `.github/workflows/Build Kernel OnePlus.yml`）

1. **不再把错误页当脚本**：`set -o pipefail`，拉取改为 `curl -fsSL -o /tmp/ksu_setup.sh` + `bash <file>`。
   404/网络错误会立刻以明确信息失败，而不是变成 127。
2. **管理器分支预检**：raw 返回非 200 时，打印 `pershoot/KernelSU-Next` 的全部可用分支再退出（exit 12）。
3. **脚本内容校验**：`bash -n` 检查下载内容，防住 HTML 错误页/截断（exit 13）。
4. **第三段校验**：必须是 7–40 位十六进制提交 hash；`Numbersf` 这类旧格式标识符会直接报错（exit 11），
   而不是被当作 commit hash 传给 `git checkout`。
5. **内置分支真正生效**：先查官方 `KernelSU-Next/KernelSU-Next`，不存在则从 `pershoot/KernelSU-Next`
   补取，两边都没有则报错（exit 14）。
   此前选 `dev-susfs` 时官方仓库并没有这个分支，`setup.sh` 会**静默**回退到默认分支，等于设置被忽略。
6. **版本号计算去掉 `expr`**：API 限流拿不到 `Link` 头时安全按 0 处理；`kernel/Kbuild` 中不存在对应变量时
   给出 warning，而不是让 `sed` 因找不到文件失败。
7. **补全 `KSU_META` 注释与描述**，写明可用取值，避免再填旧格式。

## 用法

`KSU_META` 格式：`管理器分支/内置分支/[提交hash]`

| 示例 | 含义 |
| --- | --- |
| `dev/dev-susfs/` | 默认。pershoot 的 susfs 分支 |
| `dev/dev/` | 官方 `dev` 分支 |
| `dev/next-susfs/` | pershoot 的 next-susfs 分支 |
| `dev/dev-susfs/1a2b3c4d` | 指定该分支下的某个提交 |

管理器分支必须在 `pershoot/KernelSU-Next` 中存在：
`dev`、`dev-susfs`、`next-susfs`、`test` 等（报错信息里会自动列出）。

第三段留空表示使用分支最新提交；**不要再填旧的版本标识符**。

## 验证状态

- YAML 解析通过（54 steps / 22 inputs）
- 步骤脚本 `bash -n` 通过
- 网络侧逻辑实测：
  - `dev` → HTTP 200；`main` → curl exit 22，并列出 10 个可用分支
  - `dev-susfs`：官方不存在、pershoot 存在 → 走 fork 回退分支
  - `KSU_VERSION` 解析 = 33397，`KSU_GIT_TAG` = v3.4.2
- **未跑完整构建**（一次完整内核编译约 1 小时），首次 dispatch 仍建议先看前 25 步是否全部变绿
