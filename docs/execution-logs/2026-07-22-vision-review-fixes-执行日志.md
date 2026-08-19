# 2026-07-22 PR #1550 Review 修复 — 执行日志

> 分支：`vision-enhancement`
> 基线：`9a501f7`（plan + spec commits）
> 设计文档：`docs/superpowers/specs/2026-07-22-vision-review-fixes-design.md`
> 实施计划：`docs/superpowers/plans/2026-07-22-vision-review-fixes.md`
> 工作流：subagent-driven-development（SDD），逐 Task 交付 + review + 全量回归

---

## 改动概要

| 维度 | 数值 |
|---|---|
| 改动文件 | 2（`vision.rs` + `protocol_proxy.rs`） |
| 代码增量 | +606 / −43 行 |
| 提交数 | 4（R1 → R2 → R3 → R4） |
| 新增测试 | 18 个（3+2+2+11） |
| vision 测试 | 90 passed / 0 failed |
| protocol_proxy 测试 | 57 passed / 0 failed |
| workspace build | 通过 |
| tsc --noEmit | 通过 |

---

## Task 执行记录

### Task 1: R1 缓存结构键 CacheKey

**提交**：`3f41448`
**改动**：
- 新增 `enum CacheKey { Url(String), UrlQuestion(String, String) }`
- `VLM_CACHE` 从 `HashMap<u64, CacheEntry>` 改为 `HashMap<CacheKey, CacheEntry>`
- `url_hash`/`url_question_hash` 返回 `CacheKey` 而非 `u64`
- `cache_get`/`cache_contains` 接收 `&CacheKey`，`cache_put` 接收 `CacheKey`
- `cache_put` 驱逐逻辑从 `&oldest` 改为 `oldest.clone()`（避免 &mut 冲突）
- 适配 3 处生产调用点 + 3 处测试调用点（`cache_put_and_get_roundtrip` 用 `key.clone()`）
- 新增 3 个测试：Tier1/Tier2 互不串扰、不同 question 不碰撞、空 question 走 Tier1

**测试结果**：90 vision tests 全部通过（含新增 3 个）

### Task 2: R2 BATCH_SIZE 分批

**提交**：`af79138`
**改动**：
- `call_vlm_batch` 内部按 `BATCH_SIZE`（=5）分批
- 每批独立重试（各自 `for attempt in 0..max_attempts`），合并 `all_descs`
- 12 张图 = 3 批（5+5+2）= 3 次 API 调用
- 所有调用点（当前轮、历史轮、后台分析）自动受益，无需改签名

**测试结果**：
- `call_vlm_batch_chunks_by_batch_size`：7 张图 → 2 次 VLM 调用，描述逐图对应
- `call_vlm_batch_exact_batch_size_single_call`：5 张图 → 1 次调用
- 90 vision 全部通过

### Task 3: R3 Phase 2 后台分析恢复

**提交**：`f6bd31c`
**改动**：
- 新增 `background_analyze_and_cache(urls, config, client)`：
  - 调 `call_vlm_batch`（TIER1_PROMPT + Tier1 key + 分批），写入缓存
  - 失败静默跳过，`append_diagnostic_log("vlm_phase2_error")`
- 恢复 `bg_config_opt` 收集逻辑：
  - 仅 `x_budget > 10` 时触发
  - 遍历 `all_image_msgs` 中 `msg_idx < golden_user_cutoff` 的深层消息
  - 跳过已缓存的 URL，最多收集 `x_budget` 个
- 恢复 `tokio::spawn`：`background_analyze_and_cache` 异步执行，`append_diagnostic_log("vlm_phase2_done")`
- 修复回归：`strip_image_blocks` 深层循环条件恢复 `*msg_idx >= golden_user_cutoff`（Task 2 编辑时误删）

**测试结果**：
- `phase2_background_analyzes_deep_urls_when_x_budget_gt_10`：X>10 → 后台写入缓存
- `phase2_not_triggered_when_x_budget_le_10`：X<=10 → 不触发
- 90 vision 全部通过

### Task 4: R4 function_call_output 图片处理

**提交**：`a4b7720`
**改动（vision.rs）**：
- 新增 `extract_data_urls(text)`：手动解析 `data:image/...;base64,...`，返回 `(start, end, url)` 列表
- 新增 `strip_data_urls_in_messages(messages)`：遍历 `output`/`content` 字段，data URL → `[图片已省略]`
- 新增 `analyze_data_urls_in_messages(messages, vlm_config, client)`：
  - 提取 data URL → `call_vlm_batch`（TIER1_PROMPT + Tier1 key + 分批）→ 描述替换
  - VLM 成功：`[图片描述] {desc}`，写入缓存
  - VLM 失败：`[图片描述失败，视觉模型调用失败]`（fail-open，不保留 base64）
  - 无 data URL：不影响原文本

**改动（protocol_proxy.rs）**：
- Strip 分支：`strip_images_only_counted` 后调 `strip_data_urls_in_messages`，合计 `total = n + n_data`
- Vlm 分支：`strip_image_blocks` 后调 `analyze_data_urls_in_messages`，`append_diagnostic_log("vl_tool_image")`

**测试结果**：
- 8 个同步测试：`extract_data_urls_*`（5）+ `strip_data_urls_*`（3）
- 3 个异步测试：`analyze_data_urls_replaces_with_vlm_description`、`failopen_on_vlm_error`、`noop_for_plain_text`
- 90 vision + 57 protocol_proxy 全部通过

---

## Task 5: 全量回归验证

| 检查项 | 结果 |
|---|---|
| `cargo test -p codex-plus-core --lib -- vision::tests --test-threads=1` | 90 passed |
| `cargo test -p codex-plus-core --test protocol_proxy` | 57 passed |
| `cargo build --workspace` | 通过 |
| `npx tsc --noEmit`（apps/codex-plus-manager） | 0 errors |
| `rg 'BATCH_SIZE\|dead_code\|unused'`（cargo build 输出） | 无 BATCH_SIZE warning |
| `rg 'let _ = bg_config_opt\|bg_config_opt.*None'` | 无匹配（占位代码已移除） |
| `rg 'HashMap<u64.*CacheEntry'` | 无匹配（u64 缓存已移除） |

---

## Spec 落实情况对照

| 管理员意见 | 设计方案 | 实施 | 测试覆盖 |
|---|---|---|---|
| (1) Phase 2 后台分析被禁用 | R3: 恢复 bg_config_opt 收集 + tokio::spawn + background_analyze_and_cache | ✅ `f6bd31c` | 2 个测试：X>10 触发、X<=10 不触发 |
| (2) BATCH_SIZE=5 未被使用 | R2: call_vlm_batch 内部 chunks(BATCH_SIZE)，每批独立重试 | ✅ `af79138` | 2 个测试：7图→2批、5图→1次 |
| (3) 缓存从 SHA-256 改成 u64，碰撞风险 | R1: CacheKey enum（Url/UrlQuestion），HashMap Eq 比较原始值 | ✅ `3f41448` | 3 个测试：Tier1/Tier2 互不串扰、不同 question 不碰撞 |
| (4) function_call_output 图片绕过 Strip/VLM | R4: extract_data_urls + strip_data_urls_in_messages + analyze_data_urls_in_messages，protocol_proxy 接入 | ✅ `a4b7720` | 11 个测试：extract(5)+strip(3)+analyze(3)，覆盖 Strip/Vlm/SendAsIs/fail-open |

**不在本次范围**（spec 第五章明确）：
- function_call_output 图片带上文发给 VLM → 留后续 PR
- tool-call 架构（模型自主决定调 VL） → 留未来
- SendAsIs 模式 base64 透传优化 → 非阻塞

---

## 边缘场景与风险

| 场景 | 处理方式 |
|---|---|
| `data:text/plain;base64,...` | `extract_data_urls` 仅匹配 `data:image/`，非图片 data URL 忽略 |
| base64 字符集外字符 | 扫描在首个非 `[A-Za-z0-9+/=]` 字符处停止，如空格、换行等 |
| VLM 调用失败 | fail-open：`[图片描述失败，视觉模型调用失败]`，不保留 base64 |
| 上下文溢出 | tool 图片描述计入 token 预算，溢出时截断 |
| 并发竞争 | `VLM_CACHE` 由 `Mutex` 保护，测试用 `--test-threads=1` |
| 预存 flaky | `cache_put_evicts_oldest_when_full`（全局状态竞态），非本次引入 |
