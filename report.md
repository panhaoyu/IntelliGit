# Row activation gestures select row text

#### 现象

在提交图中按住 Shift 点击多行时，行被选中的同时，浏览器还会把上一锚点到当前点击位置之间的文本一起选中。截图里表现为提交信息、版本徽标、作者/日期被蓝色文本选区覆盖，和行选中背景叠在一起，显得很脏。

在提交详情的 Changed Files 中双击文件名打开差异时，浏览器也会先选中双击位置的文件名，再触发打开差异。

#### 复现步骤

- 打开 `IntelliGit: Show Git Log`
- 点击第一行提交
- 按住 Shift 点击第三行提交
- 观察三行已被选中，但 `window.getSelection().toString()` 不为空

修复前的 E2E 断言输出：

```text
Expected: ""
Received: "ash entry two
1
index on main: 4d7fe713 Merge branch 'topic/mergeable'
Conflicting "
```

同时 `aria-pressed="true"` 的行数为 3，说明多选本身是正常的，错的是浏览器原生文本选区。

#### 根因

`.commit-row` 是带 `role="button"` 的列表行，但样式里没有关闭文本选择。Shift+点击会触发浏览器原生的“从上次点击位置扩展文本选区”，所以选中多行的同时选中了行内文字。

分支面板 `src/webviews/react/branch-column/styles.ts` 的 `PANEL_STYLE` 已经设置了 `userSelect: "none"`，提交行缺少同样的处理。

文件行同理：`TreeFileRow` 是带 `role="treeitem"` 的列表行，双击是打开差异的手势，但行样式没有关闭文本选择，所以双击时浏览器先选中文件名。

#### 修复

- 提交行：在 `src/webviews/react/commit-list/styles.ts` 的 `.commit-row` 规则中加入 `user-select: none`
- 文件行：在 `src/webviews/react/shared/components/FileTreeRows.tsx` 的 `TreeFileRow` 上加入 `userSelect="none"`
- 两者都匹配分支面板的既有做法：列表行是操作目标，不是文本表面

提交信息仍可在提交详情面板中选中复制，提交行的右键菜单仍提供 Copy Revision。

#### 证据

- 提交行修复前截图：`before.png`
- 提交行修复后截图：`after.png`
- 文件行修复前截图：`file-before.png`
- 文件行修复后截图：`file-after.png`
- 提交行修复前行为：文本选区非空，内容见上面的 `Received`
- 提交行修复后行为：文本选区为空，三行仍保持选中
- 文件行修复前行为：双击 `feature3.txt` 后文本选区为 `"feature3"`
- 文件行修复后行为：文本选区为空，差异正常打开

#### 测试

新增 E2E：`tests/e2e/rowTextSelection.spec.ts`

- 提交行：打开 Git Log，点击第一行，Shift+点击第三行；断言三行 `aria-pressed="true"`，且 `window.getSelection().toString() === ""`
- 文件行：选中 `Feature commit 3`，在 Changed Files 中双击 `feature3.txt`；断言文本选区为空，差异正常打开

运行命令：

```bash
bun run build
npx playwright test --config playwright.e2e.config.ts --project=e2e tests/e2e/rowTextSelection.spec.ts
```

修复前：文件行用例 failed，选中文本为 `"feature3"`
修复后：4 passed（fixture setup、两个用例、teardown）

新增单测：`tests/webview/unit/commitRowTextSelection.test.ts`

- 断言注入的 `.commit-row` 规则包含 `user-select: none`
- 结果：1 passed

其他验证：

- `bun vitest run tests/webview/unit`：57 个文件、488 项通过
- `bun run typecheck`：通过
- `bun run lint:strict`：通过
- `bun run format:check`：通过
- `bun run architecture:check`：通过
- `bun run verify:manifest`：通过
- 本地 dev 构建：`0.43.19-dev.1`，生成 `intelligit-0.43.19-dev.1.vsix`，`package:verify` 通过（88 entries，2.21 MB）
- 已安装到本机 VS Code：`maheshkok.intelligit@0.43.19-dev.1`

版本策略：不占用上游正式版本号，使用 `-dev.N` 预发布后缀；CHANGELOG 条目放在 `[Unreleased]`，由上游维护者决定最终发布版本。

注意：本地 Playwright MCP 会在项目根目录生成 `.playwright-mcp/` 日志，该目录未被 `.vscodeignore` 排除；打包前必须删除，否则会进入 VSIX 并触发 `package:verify` 失败。

#### 涉及文件

- `src/webviews/react/commit-list/styles.ts`
- `src/webviews/react/shared/components/FileTreeRows.tsx`
- `tests/e2e/rowTextSelection.spec.ts`
- `tests/webview/unit/commitRowTextSelection.test.ts`
- `tests/webview/unit/file-tree-rows-capabilities.test.tsx`（文件行样式变化后的 markup pin 更新）
