# Bugfix: 包装模型条目与上游共用 model id，DSH 选择器点击同 id 的「(modlens vision)」条目无反应

- 上游 issue：liustack/modlens#123（OPEN，2026-10-21 报告，版本 3.27.1 实测）
- 本 fork 版本：3.27.2（上游 3.27.1 + 本修复）

## 现象

DSH 模型选择器按 **model id 单独**判断「是否已选中」。modlens 的包装 provider 注册时只改显示名（加 `(modlens vision)` 后缀）与 `provider`，把上游 `id` 原样保留，于是同一 id 同时出现在普通条目与包装条目下：当用户当前模型正是该 id 时，点击它的包装孪生条目被当作「已经在这个模型上了」而**静默短路**——勾选不移动、无 toast、无日志。换一个 id 不同的包装条目则正常。

## 证据链

1. 上游 issue #123 给出 100% 稳定复现步骤与根因定位（`dsh/index.js` 的 `withVision` / `listModels` / `resolveModel` 三处均未改动 id），并给出期望修法其一：「让包装条目的 id 唯一，例如 `modlens:<providerId>:<upstream-id>`」。
2. 修复前最小复现（用与 `src/dshPlugin.test.ts` 相同的 harness 桩，断言 listModels 条目 id ≠ 上游 id、resolve/stream 往返正确）：
   ```
   × repro #123: wrapped model id collides with upstream id
     → AssertionError: expected 'deepseek-v4-flash' not to be 'deepseek-v4-flash'
   ```
3. 基线测试（修复前，上游原始代码）：全套 912 个测试中 3 个失败，逐个查明均为环境伪影、与本 bug 无关——`main.test.ts` 的 config show 失败是沙盒 `HTTPS_PROXY` 环境变量泄漏（剥离代理 env 后通过）；另两个 dshPlugin 测试（#81 超时、#116 假时钟）在全量并行下超时/抖动，单独跑均通过。

## 根因

`dsh/index.js` 的 `registerWrapper` 闭包：`withVision()` 只覆盖 `provider` 与 `inputModalities`；`listModels` 展开 `withVision(model)` 后仅改 `name`；`resolveModel` 以 `id: model` 收尾。三处合起来使包装条目的身份与上游条目在 id 维度上不可区分。

## 修复（采 issue 期望方案 1 的插件侧版本）

仅改 `dsh/index.js` 的 `registerWrapper` 闭包（+25/-5 量级）：

- 新增 `wrappedModelId` / `upstreamModelId` 一对编解码：包装 id = `modlens:<上游id>`，解码只剥一层前缀。
- `listModels`：条目 `id` 改为包装 id。
- `resolveModel`：先解码再调上游 `resolveModelInfo`；返回的 `info.id` 为包装 id。**向后兼容**：修复前会话里保存的普通 id 仍可解析，并就此迁移为包装 id；报错文案改用解码后的上游 id。
- `stream`：转发给上游前把 `options.model` 解码为普通 id，上游路由只认自己的 id；`provider`/`via`/消息转换逻辑不动。

配套测试更新（断言的是旧错误契约，随修复同步，共 4 处）：
- `src/dshPlugin.test.ts`：resolved id、prepareCall 的 model id 改为 `modlens:deepseek-v4-flash`；新增 listModels 条目 id 断言。
- `src/dshModelScope.test.ts`：自动发现包装条目的 id 期望改为 `modlens:nova-1`。

## 复验（修复后）

| 检查 | 结果 |
|---|---|
| 最小复现 | ✅ 通过（修复前失败，见上） |
| 全套测试 `vitest run` | ✅ 44 文件全过：**905 passed / 7 skipped / 0 failed**（exit 0） |
| `tsc --noEmit` | ✅ exit 0 |
| `biome check`（改动文件） | ✅ 无问题 |
| `vite build` | ✅ 产出单一 `dist/main.js`（200.62 kB），与基线一致 |

验证环境说明：测试需把 `TMPDIR` 指向磁盘目录（沙盒 /tmp 为 512MB tmpfs，写满会产生 ENOSPC 伪失败），并剥离沙盒代理 env 后跑 `main.test.ts`；两者均为本沙盒环境因素，已在基线阶段逐个证伪，与代码无关。
