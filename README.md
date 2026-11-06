# DSH Compaction Recipe

A small, copy-paste recipe for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (DSH) context compaction.

It does not replace DeepSeek's compaction engine. The engine is already included with DSH as `@deepseek-ai/dsh-compaction-basic` (MIT). This repository only documents a policy that worked well for long desktop sessions: start compacting at **20%** of the context window, keep about **5%** of recent messages verbatim, and make sure the cheap tool-result pruner cannot skip the real summary.

## Why this exists

On a 1M-token window, 20% is about 200,000 tokens. If compaction only trims a little and then stops — for example the footer goes from 20% to 18% — almost every later request still sends roughly 180k–200k tokens of history. That keeps input cost high for long chats.

The settings people expect ("compact at 20%, keep 5%") live in `compaction-basic`:

| Setting | Meaning | This recipe |
| --- | --- | --- |
| `thresholdRatio` | Compact when usage reaches this fraction of the routed context window | `0.20` |
| `retainRatio` | How much recent context to keep verbatim (cannot be combined with `retainTokens`) | `0.05` |

Going much lower than 5% (for example keeping only about 20,000 tokens) saves a bit more, but it is easier to lose details of the current task. 5% of a 1M window is about 50,000 recent tokens: enough to keep the live work intact, while older turns become a structured checkpoint.

## The pitfall: 20% drops to 18%, then nothing else happens

`compaction-basic` can use an optional companion, `@deepseek-ai/dsh-compaction-tool-result-pruner`. When usage crosses the threshold, it first shortens oversized tool results (default: rewrite results longer than 8192 characters down to a 4096-character head and a 1024-character tail).

If that pass alone brings usage back under the threshold, **the engine skips the LLM summary**. The footer then looks like a compaction happened, but the conversation was not actually condensed. Requests stay large until the next time usage crosses 20%, and the same light trim may fire again.

The real summary is the step that replaces older turns with a `<compacted-summary>` checkpoint and keeps only the recent tail described by `retainRatio`. After that, a 1M-token session often lands around 6% in the footer: the 5% retained tail, plus the summary, system prompt, tools, and whatever the current turn is still generating.

This recipe isolates the pruner slot and does **not** load the pruner plugin, so crossing 20% always runs the heavy summary down toward about 5%. The tradeoff is that huge tool results are no longer cheaply trimmed on their own; they wait until the 20% summary.

## How to use it

1. Copy [examples/cordis-compaction.patch.yml](examples/cordis-compaction.patch.yml) into your DSH overlay: `~/.dsh/cordis.patch.yml`, or a profile file such as `~/.dsh/profiles/desktop/cordis.patch.yml`.
2. If you already have a `compaction` group, merge the policy instead of pasting a second copy: set `thresholdRatio: 0.20` and `retainRatio: 0.05` on `@deepseek-ai/dsh-compaction-basic`, and remove `@deepseek-ai/dsh-compaction-tool-result-pruner` from that same isolate if you want every trigger to summarize.
3. Restart DeepSeek Harness so the overlay reloads.
4. Watch the usage meter in the composer footer. A successful heavy compaction should drop from about 20% to around 5–7%, not merely 20% to 18%.

You can also limit the policy to specific provider/model pairs with `modelPolicies`. The example uses a global default so it is not tied to one router or model name.

## What this is not

- Not a fork of DSH, and not a new compression plugin.
- Not a dump of a personal `cordis.patch.yml` (prompts, model lists, and local workflow do not belong here).
- Not a guarantee that the model will remember every early detail. Older context is summarized on purpose. The recent tail and the checkpoint (goal, files, errors, pending work, next step) are what carry the current task forward.

## License

[MIT](LICENSE)

---

## 中文说明

这是一份 DeepSeek Harness（DSH）上下文压缩的配置说明，不是压缩引擎本身。引擎是 DSH 已包含的 `@deepseek-ai/dsh-compaction-basic`。

建议把触发线设在上下文窗口的 **20%**，重压缩后原样保留最近大约 **5%** 的对话。在大约 100 万 token 的窗口里，这分别是大约 20 万和 5 万 token。比 5% 再低，省得有限，却更容易丢掉当前任务的细节。

一个容易踩的坑：如果同时启用了工具结果剪枝（`dsh-compaction-tool-result-pruner`），它会先把超长工具输出剪短。一旦用量因此掉回 20% 以下，引擎就不再做模型摘要。界面上可能只看到 20% 变成 18%，后面每一轮请求体仍然很大。这份示例故意不加载剪枝插件，让每次到达 20% 都走到重压缩，用量一般会落到大约 6%（5% 的近期原文，加上摘要、系统提示和正在生成的内容）。

把 [examples/cordis-compaction.patch.yml](examples/cordis-compaction.patch.yml) 合并进 `~/.dsh/cordis.patch.yml` 或对应 profile 的 overlay，然后重启 DSH 即可。
