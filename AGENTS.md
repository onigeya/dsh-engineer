<!-- bmad:context -->
<!-- Verified 2026-11-09 against main（仓库尚零提交；refresh 请从首个 commit 起 diff）. Managed by bmad-project-context; edits inside this block are replaced on refresh. Keep anything you want preserved outside the markers. -->

## dsh-engineer

DeepSeek Harness（DSH/Cordis）插件代码库，仓库根即一个可直接安装的 workspace bundle：npm 包名 `dsh-engineer`，MIT，ESM TypeScript。产品命题（V 层反馈传感器 + 证据门 + 阶段机薄包装）见 `_bmad-output/brainstorming/brainstorm-dsh-vibe-coding-plugin-2026-11-09/brainstorm-intent.md`。宿主源码与官方契约文档都在本仓之外，路径见下。

## Policy

- `_bmad/`、`.agents/` 为 BMAD 工具链，永不提交；`_bmad-output/` 规划产物必须提交（项目知识）。
- 永不手改 active profile 的 `package.json` / `cordis.patch.yml` / `pnpm-workspace.yaml`（Host-owned，位于 `C:\Users\ellio\.dsh\profiles\web\`）；一切变更打成 bundle 后经 `plugin_manager install_bundle`（bundle 目录绝对路径）安装。
- 不往 `$DSH_HOME` 下放自建包，不在 profile 目录内跑 `pnpm install`。

## Where things are

- 宿主源码（agent-loop 主脉）：`D:\workspace\deepseek-harness\packages\core\`；可发行组合范例：`packages\bundle\`
- 官方插件开发文档：`D:\workspace\deepseek-harness\docs\user\develop\`；逐服务契约：`docs\subsystems\*.md`——generated Cordis API 段与源码校验过，优先于散文；写事件前先读 `docs\cordis-primer.md`
- 已装发行版 `D:\DeepSeek Harness\resources\app\dsh\` 为只读参考：仅宿主文件工具可读，shell 工具读它会失败——这不是文件不存在
- 开发流程技能：`dsh-plugin-dev`（本机 `C:\Users\ellio\.dsh\skills\`）

## Running and verifying

- 本仓暂无 `package.json` 与任何脚本——bootstrap 后补 typecheck / lint / build 门禁命令；build 脚本必须让 `tsc` 失败以非零退出（tsc 报错仍 emit 产物）。
- 开发回环在宿主 checkout 里跑：`pnpm dsh web --patch <cwd 相对的 patch 路径>`；patch 行内 `name:` 必须写绝对路径——loader 按 profile 目录解析相对路径。
- 装完必须用 `cordis_inspect_query`（Service/Event/Config/Tool/Slots）验证挂载；`plugin_manager list_plugins` 只报安装请求不报生效，`restart-required` 要重载 GUI 再看。
- 版本：Node ≥ 24.9；本机 pnpm 12.x（技能文档里的「pnpm 11」过时）；bundle 声明 `engines.dsh >=0.2.0`。

## Conventions that differ from defaults

- 插件 = 导出 `name` / `inject` / `apply` 的 TS 模块；一切注册走 `ctx.effect` / `ctx.on` / `register()`——手写 removeListener、clearInterval 是红线（卸载回收与 HMR 的前提）。
- waterfall 事件监听器必须调 `next()`，漏调即短路下游。
- Config 必须 Standard Schema；参数不许硬编码进代码——判据：能否只改 cordis.yml 不改代码。
- patch 层按行 `id` 整段替换 config，没有深合并——想保留的键必须整行重述。
- 不提前拆能力缝：Definition/Provider/Consumer 三层只在角色确需独立演化时才分包；Provider 与 Consumer 互不依赖、只依赖 Definition。
- 从 git 直装拉的是源码、不会跑 build：要么写自包含 `prepare`，要么发布构建好的 `lib/` 或 tarball。

## Known pitfalls

- 多个异步 disposer 并发执行、无串行保证——有顺序依赖的清理收进同一个 `ctx.effect()` 的单个 disposer。
- UI 展示方法（presentCall/presentResult）在日志回放时也会执行：禁 I/O、会话状态、时钟、随机；且内置 Web Client 不消费这两个方法——Client 插件把 wire 工具名注册进 `tool.call.toolview` 的 keyed 槽。

<!-- /bmad:context -->
