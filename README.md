# subtitle-generator

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/yuka9611/subtitle-generator/blob/main/subtitle_generator.ipynb)

Google Colab 字幕生成 Notebook。`subtitle_generator.ipynb` 是唯一执行源码；脚本只用于保持 Notebook 序列化、静态检查和合成验证，不能替代 Colab 运行顺序。

## 当前流程

```text
输入媒体
  -> FFmpeg 16 kHz mono PCM audio_reference.wav
  -> Silero CPU activity / 自然边界 / 风险诊断
  -> 可选 ClearVoice 独立子环境 WAV A/B
  -> MOSS-Transcribe-Diarize ASR + speaker + 粗时间戳
  -> 严格 parser -> NormalizedSegment
  -> whole/chunk 去重 -> 证据图 speaker stitching
  -> Qwen3-ForcedAligner 原生 Transformers 对齐
  -> words 通用后处理 -> SRT/ASS + diagnostics ZIP
```

`audio_reference.wav` 是全局时间基准。`VAD_AUDIO_FILE` 和 `ALIGN_AUDIO_FILE` 永远指向 reference；`MOSS_AUDIO_FILE` 默认也指向 reference，只有通过 ClearVoice 采样率、长度漂移和基础质量门槛后才允许替换。ClearVoice 默认关闭，子环境通过 WAV 交换，不能改变 reference 时间轴。

普通 UI 只显示四个输入：

| 输入 | 默认值 | 作用 |
|---|---|---|
| 语言 | `ja` | prompt 与对齐诊断提示 |
| 热词预设 | `原神` | 生成 prompt 中的热词提示 |
| 自定义热词 | 空 | 只以 hash 进入 fingerprint，明文不进入 compact 产物 |
| 预期说话人数 | 空 | 仅用于诊断，不限制 MOSS 说话人数量 |

speaker ID 与 ASS 样式动态生成，覆盖 2、3、4、5–6、8+ 人。whole-audio 优先；当前 85 分钟是待新鲜 T4 基准确认的候选上限，不是已验收承诺。超限、OOM 或已知截断时使用初始 45 分钟块、75 秒重叠，并在 45 秒范围内搜索 Silero 自然边界。分块去重与 speaker mapping 分离；mapping 使用时间 IoU、重叠文本和轮次上下文证据，确定性 Hungarian 证据不足时创建新的 uncertain global speaker，绝不按局部数字直接拼接。

MOSS 使用官方 prompt 结构和 greedy decoding，显式记录 `prompt_len`、`generated_tokens`、`max_new_tokens` 与 stop reason。严格解析 `[start][Sxx]text[end]`，保留合法记录并把 malformed、空文本、逆序、越界、重复和零时长写入诊断；没有有效片段或尾部明确不完整时硬失败。rescue v1 只产出诊断候选，不能写入正式字幕。

Qwen3-ForcedAligner 使用同一主环境的原生 Transformers API，只读 reference；单任务约束在 240 秒以内，失败、零 span 或越界回退 MOSS 粗时间戳并标记 fallback。后处理统一消费 `words` 字段。没有独立文本、独立时间证据和正时长时，重叠语音只保留 metadata，不复制主字幕到第二条 ASS Dialogue。

## 运行产物与状态

每次运行使用 `/content/subtitle-generator-runs/<run_id>/` 和输入/reference hash、语言、热词 hash、模型 revision、VAD、长音频与增强配置构成的 `run_fingerprint`。所有阶段使用统一 envelope：`schema_version`、`run_id`、`run_fingerprint`、`stage`、`status`、`created_at`、`summary`、`metrics`、稳定 ID 的 warnings/errors 和 artifacts。JSON 采用 `.tmp` 写入后 rename。

必须检查的产物包括：`run_manifest.json`、`diagnostics_index.json`、`audio_preprocess_diagnostics.json`、`vad_activity.json`、`vad_diagnostics.json`、`moss_raw_output.txt`、MOSS raw/normalized/diagnostics/rescue 文件、`aligned_segments.json`、`alignment_diagnostics.json`、`speaker_stitch_diagnostics.json`、`final_items.json`、gap/postprocess/export diagnostics、`output.srt`、`output.ass`、summary JSON/Markdown 和两个 diagnostics ZIP。

compact ZIP 只允许 `compact_safe` 元数据，不含 transcript、媒体、绝对私有路径、自定义热词明文或 token/cookie。private ZIP 可包含字幕和分段文本，但默认排除原始及增强媒体。`partial`、`fallback`、`failed`、`skipped` 必须传播到摘要、导出和 benchmark；不能伪装成 clean。

## 使用与验证边界

1. 在新鲜 Colab T4 中按 Notebook 顺序执行；模型阶段严格串行：Silero CPU → 可选 ClearVoice 子进程 → MOSS GPU → Qwen GPU → 后处理。
2. 本地每次 Notebook 修改后运行 `python3 -m json.tool subtitle_generator.ipynb`、去除 magics/shell 后的全部 code cell compile、粒度统计和旧关键词零残留检查。
3. 真实验收覆盖短反应、重叠语音、长静音、尾音、30–60、60–85、90–120 分钟、OOM/截断、malformed parser、stale state、Qwen zero span 和 ClearVoice drift；指标至少包含 CER、cpCER、ASS coverage、DER、speaker-change F1、token-speaker accuracy、overlap F1、short-reaction recall、fragmentation/false merge 与资源用量。
4. 本机没有可用 Colab T4 和私有样本，因此本仓库内的 JSON/AST/合成结果不代表真实模型质量、T4 显存、长音频尾部覆盖或 ClearVoice/Qwen 实际可用性。

架构细节见 [docs/architecture.md](docs/architecture.md)，静态与 Colab 验收见 [docs/runbooks/notebook-validation.md](docs/runbooks/notebook-validation.md)，决策记录见 [docs/decisions/ADR-0003-moss-native-pipeline.md](docs/decisions/ADR-0003-moss-native-pipeline.md)。
