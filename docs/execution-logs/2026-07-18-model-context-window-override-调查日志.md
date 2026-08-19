# 调查日志：xf-yun 供应商模型窗口统一识别为 256k

- 日期：2026-07-18
- 关联会话：`019f743a-7480-7172-92ba-c64c13204df4`（xf-yun，模型 `xopdeepseekv4flash`）
- 对照会话：`019f7439-0214-7da1-9291-cfeeeeedbf4e`（volces，模型 `deepseek-v4-flash`）
- 方法：systematic-debugging（先定位根因，再给方案，本次不改代码）

## 一、问题描述

- 供应商 `https://maas-coding-api.cn-huabei-1.xf-yun.com/v2`（经 Codex++ 中转 `http://127.0.0.1:57321/v1`）下的模型，上下文窗口被 Codex 统一识别为 **256k**。
- 用户已在 Codex++ 里按模型标记了窗口（`xopdeepseekv4flash` = 1000k 等），但生效值仍是 256k。
- 对照：另一供应商 `https://ark.cn-beijing.volces.com/api/coding/v3`（volces），无论 chat 还是 responses 格式，标记的 1000k 都能被正确识别。

## 二、现象与证据（A/B 对照）

同一天、同一 Codex 二进制（`/Applications/ChatGPT.app/Contents/Resources/codex`，`cli_version 0.145.0-alpha.18`）、同一 Codex++ 版本，唯一变量是 `config.toml` 是否写了顶层 `model_context_window`。

| 维度 | xf-yun（异常 ✗） | volces（正常 ✓） |
| --- | --- | --- |
| 当前 `~/.codex/config.toml` 顶层 `model_context_window` | `= 256000`（**存在**） | **不存在** |
| `model_catalog_json` 指向的 catalog 里该模型的 `context_window` | `xopdeepseekv4flash` = **1000000** ✓ | `deepseek-v4-flash` = **1000000** ✓ |
| 会话 `task_started` 事件里 Codex 报告的 `model_context_window` | **256000** ✗ | **1000000** ✓ |

证据命令与结果：

```bash
# xf-yun 会话：Codex 报告 256000
$ grep -oE '"model_context_window":[0-9]+' .../rollout-...-019f743a-...-c64c13204df4.jsonl | sort | uniq -c
   6 "model_context_window":256000

# volces 会话：Codex 报告 1000000
$ grep -oE '"model_context_window":[0-9]+' .../rollout-...-019f7439-...-cfeeeeedbf4e.jsonl | sort | uniq -c
   2 "model_context_window":1000000
```

```bash
# xf-yun catalog 里该模型窗口本就是对的
$ python3 -c "..."  # 读 ~/.codex/model-catalogs/relay-mr95sjrq.json
  slug='xopdeepseekv4flash'  context_window=1000000

# volces 最近一份 backup config 顶层没有 model_context_window
$ grep -c "model_context_window" ~/.codex/backups/codex-plus-live-1784249764461/config.toml
0
# xf-yun live config 有
$ grep -c "model_context_window" ~/.codex/config.toml
1
```

结论：catalog 两侧都对（1000k），但 xf-yun 的 `config.toml` 多了一行 `model_context_window = 256000`，Codex 用它覆盖了 catalog。

## 三、根因

**Codex 把顶层 `model_context_window` 当作「全局覆盖（override）」，而非「回退默认（fallback）」。** 一旦 `config.toml` 写了 `model_context_window`，它就盖掉 `model_catalog_json` 里所有逐模型 `context_window`，包括用户已正确标记的 1000k。

而 Codex++ 的 `RelayProfile.context_window`（UI「更多选项 → 上下文大小」字段）会被原样写进 `config.toml` 顶层 `model_context_window`。xf-yun 这个 profile 的该字段填了 `256000`，于是 256000 覆盖了 catalog 的 1000k；volces 那个 profile 该字段留空，Codex++ 不写 `model_context_window`，Codex 才回退到 catalog，正确得到 1000k。

> 注意：这与 Codex++ 设计文档的措辞不一致。`docs/research/01-调研结果.md:65,79` 把 `model_context_window` 描述为「未单独配置时的回退默认」，但实测 Codex 行为是「全局覆盖」。这是本次问题的认知根。

### 数据流（证据链）

1. UI 输入 → profile 字段
   - `apps/codex-plus-manager/src/App.tsx:4456` 「上下文大小」输入框，placeholder「留空不改写，例如 200000」（`App.tsx:4458`）。
   - `App.tsx:6960` `setRootTomlIntKey(configContents, "model_context_window", patch.contextWindow || "")`：把字段值写进 profile 的 `configContents`。

2. profile 字段 → config.toml 顶层
   - `crates/codex-plus-core/src/relay_config.rs:1494` `apply_context_limits_to_config`：
     ```rust
     if let Some(value) = parse_optional_positive_u64(context_window, "上下文大小")? {
         doc["model_context_window"] = toml_edit::value(value as i64);
     }
     ```
     仅当 `profile.context_window` 非空正整数时才写 `model_context_window`；留空则不写（也不主动删除已有的，但基座来自 `complete_relay_profile_config(profile)`，前端已用 `setRootTomlIntKey` 删掉，故留空 = 不出现）。

3. 逐模型窗口 → catalog（这条链是对的）
   - `relay_config.rs:1508` `apply_model_catalog_to_config` → `model_suffix.rs:203` `build_model_catalog_json`：用 `model_windows` map 的逐模型值生成 catalog；无后缀条目用 `fallback = profile.context_window`（`relay_config.rs:1547`）。
   - 实测 xf-yun catalog 里 `xopdeepseekv4flash` = 1000000，说明用户的逐模型标记已正确进 catalog。

4. Codex 运行时读取：`config.toml` 有 `model_context_window` → 用它（256000），忽略 catalog；没有 → 用 catalog（1000000）。

### 256000 从哪来

- 不是代码字面量（全仓搜 `256000/256_000` 无命中）。
- `provider_import.rs:207`、`ccs_import.rs:97` 导入时 `context_window: String::new()`（空），不是导入塞的。
- `relay_config.rs:1606` 反向读取：`profile.context_window = root_positive_int_string(config_text, "model_context_window")`——一旦 `config.toml` 里曾出现过 `model_context_window`（旧版本写入 / 手改 / 其他同步），Codex++ 会把它「认领」回 profile 字段并持续回写，形成粘性值。
- 最可能：xf-yun profile 的「上下文大小」字段被填过 `256000`（可能误填、或从某个旧值认领），之后一直被回写覆盖 catalog。

## 四、为什么 volces 没问题

volces profile 的「上下文大小」字段为空 → Codex++ 不写顶层 `model_context_window` → Codex 回退到 `model_catalog_json` → 读到 `deepseek-v4-flash` = 1000000。所以 chat / responses 两种格式都正常（context window 与 wire_api 无关，仅由 config 解析决定）。

## 五、解决方案

### 5.1 立即修复（用户侧，不改代码）

在 Codex++ 里把 **xf-yun profile 的「更多选项 → 上下文大小」字段清空**（留空），保存/应用。

- 前端 `App.tsx:7095` `setRootTomlIntKey`：值为空时调用 `removeRootTomlKey`，会把 `model_context_window` 从 `configContents` 删除。
- 后端 `relay_config.rs:1499`：字段空时不写 `model_context_window`，基座里又已被前端删掉，故 `config.toml` 最终不再有这一行。
- 之后 Codex 回退到 catalog，`xopdeepseekv4flash` 识别为 1000000。

> 如果同 profile 下还有别的模型想用不同窗口（如 500k），继续靠「逐模型窗口」列维护即可——那走 catalog，不被覆盖。**只要「上下文大小」这个全局字段留空，逐模型窗口才会生效。**

### 5.2 验证步骤

1. 清空字段并应用后，检查 `~/.codex/config.toml`：
   ```bash
   grep -c "^model_context_window" ~/.codex/config.toml   # 期望 0
   grep model_catalog_json ~/.codex/config.toml           # 仍存在
   ```
2. 新开一个 Codex 会话，检查 `task_started` 事件：
   ```bash
   grep -oE '"model_context_window":[0-9]+' ~/.codex/sessions/2026/07/18/rollout-*.jsonl | sort | uniq -c
   # 期望出现 "model_context_window":1000000
   ```
3. 回到 Codex++ 标记处确认窗口显示为 1000k。

### 5.3 后续改进建议（仅记录，本次不改代码）

1. **修正文档认知**：`docs/research/01-调研结果.md:65,79` 把 `model_context_window` 标为「回退默认」与 Codex 实际「全局覆盖」行为不符，建议改为「全局覆盖；留空时才回退到 catalog」，避免再踩。
2. **UI 加护栏**：当 profile 已配置逐模型窗口（`model_windows` 非空）时，若用户在「上下文大小」填了值，应提示「该值会覆盖所有逐模型窗口」，或在该场景下自动留空 / 禁用该字段。当前 placeholder「留空不改写」语义正确但藏在「更多选项」里，易被忽略。
3. **可选**：`apply_context_limits_to_config` 在 `profile.context_window` 为空时，显式移除已存在的 `model_context_window`（而非依赖前端删 + 基座重建），避免任何路径下的粘性残留。

## 六、关键文件索引

- `~/.codex/config.toml`（live）：`model_context_window = 256000`、`model_catalog_json = "model-catalogs/relay-mr95sjrq.json"`
- `~/.codex/model-catalogs/relay-mr95sjrq.json`：xf-yun catalog，`xopdeepseekv4flash` = 1000000（正确）
- `~/.codex/model-catalogs/relay-mqgvyjzg.json`：volces catalog，`deepseek-v4-flash` = 1000000
- `~/.codex/backups/codex-plus-live-1784249764461/config.toml`：volces 最近 config，无 `model_context_window`
- `crates/codex-plus-core/src/relay_config.rs:1494`（写 `model_context_window`）、`:1508`（写 catalog）、`:1606`（反向认领）
- `crates/codex-plus-core/src/model_suffix.rs:203`（catalog 构建）
- `apps/codex-plus-manager/src/App.tsx:4456`（「上下文大小」字段）、`:7095`（空值删除 key）、`:6960`（patch 写入）
- `docs/research/01-调研结果.md:65,79`（设计措辞需修正）
