# 仓库指南

- 仓库：https://github.com/openclaw/openclaw
- GitHub issues/PR 评论：使用原生多行字符串或 `-F - <<'EOF'`（或 `$'...'`）实现换行，不要嵌入 `"\\n"`。

---

## 项目结构

| 目录 | 说明 |
|------|------|
| `src/` | 源码（CLI 配线在 `src/cli`，命令在 `src/commands`，Web provider 在 `src/provider-web.ts`，基础设施在 `src/infra`，媒体管道在 `src/media`） |
| `*.test.ts` | 测试文件与源码并列放置 |
| `docs/` | 文档（图片、队列、Pi 配置），构建输出在 `dist/` |
| `extensions/*` | 插件/扩展（workspace 包），插件专用依赖放在扩展的 `package.json` 中，除非核心也需要，否则不要加到根 `package.json` |
| `../openclaw.ai` | 安装脚本仓库（`public/install.sh` 等） |

### 消息通道

重构共享逻辑（路由、白名单、配对、命令限制、引导、文档）时，需同时考虑所有内置和扩展通道：

- 核心通道文档：`docs/channels/`
- 核心通道代码：`src/telegram`, `src/discord`, `src/slack`, `src/signal`, `src/imessage`, `src/web`（WhatsApp web）, `src/channels`, `src/routing`
- 扩展通道插件：`extensions/*`（如 `extensions/msteams`, `extensions/matrix`, `extensions/zalo` 等）

添加通道/扩展/应用/文档时，检查 `.github/labeler.yml` 确保标签覆盖。

---

## 文档规范（Mintlify）

- 托管于 docs.openclaw.ai
- 内部链接：根目录相对路径，无 `.md`/`.mdx` 后缀，例如 `[Config](/configuration)`
- 章节锚点：根目录相对路径 + 锚点，例如 `[Hooks](/configuration#hooks)`
- **标题避免**：em dash（—）和撇号（'），会破坏 Mintlify 锚点链接
- README（GitHub）：保持绝对 URL（`https://docs.openclaw.ai/...`）
- 文档内容通用化：不使用个人设备名/主机名/路径，使用占位符如 `user@gateway-host`

### 中文文档（zh-CN）

- `docs/zh-CN/**` 为生成文件，除非明确要求，否则不要编辑
- 流程：更新英文文档 → 调整术语表（`docs/.i18n/glossary.zh-CN.json`）→ 运行 `scripts/docs-i18n` → 仅在指示时应用针对性修复
- 翻译记忆库：`docs/.i18n/zh-CN.tm.jsonl`（生成）
- 详情见 `docs/.i18n/README.md`

---

## 开发命令

```bash
# 运行时基准：Node 22+（保持 Node + Bun 路径兼容）
pnpm install          # 安装依赖
prek install          # 安装 pre-commit 钩子（与 CI 检查一致）
bun <file.ts>         # 优先用 Bun 执行 TypeScript
pnpm openclaw ...     # 开发模式运行 CLI
pnpm dev              # 同上
pnpm build            # 类型检查 + 构建
pnpm tsgo             # TypeScript 类型检查
pnpm check            # Lint + 格式化（Oxlint + Oxfmt）
pnpm test             # Vitest 测试
pnpm test:coverage    # 测试覆盖率
```

### macOS 应用打包

```bash
scripts/package-mac-app.sh  # 默认当前架构
```

发布清单：`docs/platforms/mac/release.md`

---

## 编码规范

- 语言：TypeScript (ESM)，优先严格类型，避免 `any`
- 格式化：Oxlint + Oxfmt，提交前运行 `pnpm check`
- 注释：为复杂或非显而易见的逻辑添加简短注释
- 文件大小：目标约 700 LOC 以下（仅作指导，非硬性限制）
- 命名：产品/应用/文档标题用 **OpenClaw**，CLI 命令/包/二进制/路径/配置键用 `openclaw`
- 避免创建 "V2" 副本，优先提取辅助函数

---

## 测试规范

- 框架：Vitest + V8 覆盖率（70% 行/分支/函数/语句阈值）
- 命名：与源码同名 + `*.test.ts`，e2e 用 `*.e2e.test.ts`
- 修改逻辑时，推送前运行 `pnpm test` 或 `pnpm test:coverage`
- 测试工作者数：不超过 16

### 真实密钥测试

```bash
# OpenClaw only
CLAWDBOT_LIVE_TEST=1 pnpm test:live

# 包含 provider live tests
LIVE=1 pnpm test:live

# Docker
pnpm test:docker:live-models
pnpm test:docker:live-gateway
pnpm test:docker:onboard
```

纯测试添加/修复一般不需要变更日志条目，除非改变面向用户的行为。

---

## 发布渠道

| 渠道 | 标签格式 | npm dist-tag |
|------|----------|--------------|
| stable | `vYYYY.M.D` | `latest` |
| beta | `vYYYY.M.D-beta.N` | `beta`（可能不含 macOS app） |
| dev | `main` 分支最新提交（无标签） | - |

---

## 提交与 PR 规范

### Commit

```bash
scripts/committer "<msg>" <file...>  # 推荐方式，保持暂存区聚焦
```

- 遵循简洁、行动导向的提交消息（如 `CLI: add verbose flag to send`）
- 将相关更改分组，避免捆绑不相关的重构

### Changelog

- 保持最新发布版本在顶部（无 `Unreleased`）
- 发布后，升级版本并开始新的顶部章节

### PR 工作流

| 模式 | 操作 |
|------|------|
| **Review** | 仅阅读 `gh pr view/diff`，**不**切换分支，**不**修改代码 |
| **Landing** | 从 `main` 创建集成分支，合并 PR 提交（优先 rebase 保持线性历史；复杂/冲突时可用 merge），应用修复，添加变更日志（+ 感谢 + PR #），本地运行完整门禁（`pnpm build && pnpm check && pnpm test`）后提交，合并回 `main`，最后 `git switch main` |

### PR 合并清单

1. 首选 **rebase**（提交干净）或 **squash**（历史混乱时）
2. 如果 squash，将 PR 作者添加为 co-contributor
3. 添加变更日志条目（包含 PR # + 感谢）
4. 在 PR 中留下评论，说明我们所做的内容和 SHA 哈希
5. 新贡献者：运行 `bun scripts/update-clawtributors.ts` 更新 README

### 参考文档

- 提交 PR：`docs/help/submitting-a-pr.md` ([Submitting a PR](https://docs.openclaw.ai/help/submitting-a-pr))
- 提交 Issue：`docs/help/submitting-an-issue.md` ([Submitting an Issue](https://docs.openclaw.ai/help/submitting-an-issue))

---

## 简写命令

| 命令 | 行为 |
|------|------|
| `sync` | 如果工作树脏，提交所有更改（选择合理的 Conventional Commit 消息），然后 `git pull --rebase`；如有 rebase 冲突且无法解决则停止；否则 `git push` |

---

## 多 Agent 安全

| 限制 | 说明 |
|------|------|
| `git stash` | 不要创建/应用/删除，除非明确要求（包括 `git pull --rebase --autostash`） |
| `git worktree` | 不要创建/删除/修改，除非明确要求 |
| 分支切换 | 不要切换分支/检出不同分支，除非明确要求 |
| 并行运行 | 可以，只要每个 agent 有自己的会话 |
| 未识别文件 | 继续进行，专注于你的更改，仅提交那些 |

### Lint/Format 变更

- 如果暂存+未暂存的差异仅是格式化，自动解决无需询问
- 如果已请求 commit/push，自动暂存并将格式化后续包含在同一提交中（或必要时的小后续提交）
- 仅当更改是语义性（逻辑/数据/行为）时才询问

---

## vm.exe.dev 操作

```bash
ssh exe.dev              # 然后 ssh vm-name（假设 SSH 密钥已设置）
sudo npm i -g openclaw@latest  # 全局安装（需要 root）
openclaw config set ...        # 配置，确保 gateway.mode=local
pkill -9 -f openclaw-gateway || true
nohup openclaw gateway run --bind loopback --port 18789 --force > /tmp/openclaw-gateway.log 2>&1 &
```

验证：`openclaw channels status --probe`, `ss -ltnp \| rg 18789`, `tail -n 120 /tmp/openclaw-gateway.log`

---

## Agent 专项说明

| 项目 | 说明 |
|------|------|
| "makeup" | = "mac app" |
| `node_modules` | 永远不要编辑（全局/Homebrew/npm/git 安装会被更新覆盖） |
| Signal "update fly" | `fly ssh console -a flawd-bot -C "bash -lc 'cd /data/clawd/openclaw && git pull --rebase origin main'"` 然后 `fly machines restart e825232f34d058 -a flawd-bot` |
| Carbon 依赖 | 永远不要更新 |
| 依赖补丁 | 需要 `pnpm.patchedDependencies` 的依赖必须使用精确版本（无 `^`/`~`） |
| 依赖修改 | pnpm 补丁/overrides/vendored 更改需要明确批准 |
| CLI 进度 | 使用 `src/cli/progress.ts`（`osc-progress` + `@clack/prompts` spinner） |
| 状态输出 | 使用 `src/terminal/table.ts`；`status --all` = 只读/可粘贴，`status --deep` = 探测 |
| Gateway (macOS) | 仅作为菜单栏应用运行，无单独 LaunchAgent/helper 标签。重启通过 OpenClaw Mac app 或 `scripts/restart-mac.sh` |
| macOS 日志 | 使用 `./scripts/clawlog.sh` 查询 OpenClaw 子系统的统一日志 |
| SwiftUI 状态管理 | 优先使用 `Observation` 框架（`@Observable`, `@Bindable`）而非 `ObservableObject`/`@StateObject` |
| 连接提供者 | 添加新连接时，更新每个 UI 表面和文档，保持同步 |
| 版本位置 | `package.json`（CLI）, `apps/android/app/build.gradle.kts`, `apps/ios/Sources/Info.plist`, `apps/macos/...`, `docs/install/updating.md` |
| 重启应用 | "重启 iOS/Android apps" = 重建（重新编译/安装）并重新启动，不仅仅是 kill/launch |
| 设备检查 | 测试前，验证连接的真实设备（iOS/Android），再使用模拟器/模拟器 |
| iOS Team ID | `security find-identity -p codesigning -v` → 使用 Apple Development (…) TEAMID |
| A2UI bundle hash | `src/canvas-host/a2ui/.bundle.hash` 自动生成，仅通过 `pnpm canvas:a2ui:bundle` 重新生成 |
| 发布签名 | 在仓库外管理，遵循内部发布文档 |
| 发布版本 | 未经操作员明确同意，不要更改版本号；运行 npm publish/发布步骤前始终请求许可 |

### 代码风格补充

- 工具 schema 约束：避免 `Type.Union`，不使用 `anyOf`/`oneOf`/`allOf`；字符串列表用 `stringEnum`/`optionalStringEnum`，可选用 `Type.Optional(...)`
- 工具 schema：避免原始 `format` 属性名（某些验证器将其视为保留关键字）
- 会话文件：`~/.openclaw/agents/<agentId>/sessions/*.jsonl`（使用系统提示 Runtime 行的 `agent=<id>` 值，最新的除非指定特定 ID）
- SSH macOS：不要通过 SSH 重建 macOS app；重建必须在 Mac 上直接运行
- 流式回复：永远不要向外部消息表面（WhatsApp、Telegram）发送流式/部分回复
- 语音唤醒转发：命令模板应保持 `openclaw-mac agent --message "${text}" --thinking low`

---

## NPM + 1Password（发布/验证）

```bash
# 使用 1password skill；所有 op 命令必须在新的 tmux 会话中运行
eval "$(op signin --account my.1password.com)"  # 登录
op read 'op://Private/Npmjs/one-time password?attribute=otp'  # OTP
npm publish --access public --otp="<otp>"  # 从包目录发布
npm view <pkg> version --userconfig "$(mktemp)"  # 验证（无本地 npmrc 副作用）
```

发布后终止 tmux 会话。

---

## 安全与配置

| 项目 | 位置 |
|------|------|
| Web provider 凭据 | `~/.openclaw/credentials/` |
| Pi 会话 | `~/.openclaw/sessions/`（基础目录不可配置） |
| 环境变量 | `~/.profile` |

**永远不要**提交或发布真实电话号码、视频或实时配置值。在文档、测试和示例中使用明显的占位符。

发布流程：始终先阅读 `docs/reference/RELEASING.md` 和 `docs/platforms/mac/release.md`。

---

## 故障排查

- 重品牌/迁移问题或遗留配置/服务警告：运行 `openclaw doctor`（见 `docs/gateway/doctor.md`）

---

## 回复规范

- 回复 GitHub Issue/PR 时，结尾打印完整 URL
- 回答问题时，仅提供高置信度答案：在代码中验证，不要猜测
- 调查 bug 时，阅读相关 npm 依赖的源代码和所有相关本地代码后再下结论
