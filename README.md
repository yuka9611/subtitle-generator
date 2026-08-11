# subtitle-generator

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/yuka9611/subtitle-generator/blob/main/subtitle_generator.ipynb)

Google Colab 字幕生成 Notebook，主流程使用 faster-whisper、Qwen Forced Align 和自动说话人识别，导出 SRT、ASS 与诊断产物。执行源只有 subtitle_generator.ipynb；脚本和文档不能替代 Notebook 的实际执行顺序。

## 普通模式

普通参数单元只显示四个输入：

| 输入 | 默认值 | 说明 |
|---|---|---|
| 语言 | ja | 识别语言 |
| 热词预设 | 原神 | 原神、星铁、无或自定义 |
| 自定义热词 | 空 | 使用自定义预设时填写 |
| 已知说话人数 | 空 | 空值自动估计；只接受 1–4 |

普通用户不需要选择 Whisper、beam、ASS 单行、词级回退、音频预处理、重叠分离、diarization 后端、场景、chunk 或边界阈值。内部固定自动策略为：

- CUDA 总显存至少 14 GiB：large-v3 / beam 5；低于 14 GiB：large-v3-turbo / beam 3。
- 没有 CUDA 时在模型流水线开始前停止，并提示启用 Colab GPU；不会静默运行完整 CPU 流程。
- 300 秒以上音频自动采用 300 秒块、25 秒重叠、20 秒边界搜索和 180 秒最小块。
- 人数为空时使用 hybrid 自动估计；填写 1 时跳过实际 diarization 并保留统一 speaker 时间轴；填写 2–4 时同时作为 pyannote 与 NeMo 的固定人数。
- pyannote 与 NeMo 按 hybrid 互为 fallback；双方失败时保留无 speaker 主轨，不丢字幕文本。

ASR、Qwen 对齐和 diarization 使用显式的 ASR_AUDIO_FILE、ALIGN_AUDIO_FILE、DIAR_AUDIO_FILE 边界，不再通过一个可变的全局 AUDIO_FILE 串扰。主轨是唯一可靠文本来源；没有独立文本与时间轴时，重叠证据只能作为 metadata，不复制主轨字幕到副轨。

## 高级基准

Notebook 末尾有一个默认关闭的高级区域。普通顺序执行不会触发矩阵；只有填写 BENCHMARK_RUN_ID 和 BENCHMARK_SAMPLE_MANIFEST 后才运行。私有媒体、人工 ASS/RTTM、candidate 输出和诊断包写入：

/content/drive/MyDrive/subtitle-generator-benchmarks/<run_id>/

固定路线和顺序如下：

1. 阶段 A：固定 Whisper/Qwen/current hybrid，比较 raw、当前 BS-RoFormer、htdemucs_ft。
2. 阶段 B：冻结同一 raw 转录与 Qwen token，在 BGM/高重叠样本比较 NeMo clustering 与 SortFormer offline 的 raw/Demucs diarization。
3. 阶段 C：相同原音频、人数先验和 tokens，比较 NeMo clustering、NeMo MSDD、pyannote、current hybrid、SortFormer standalone、hybrid + SortFormer overlap-only。阶段 B 晋级后才追加 Demucs diar stem hybrid。
4. 完整节目：只比较 current NeMo chunk/stitching、current hybrid、SortFormer streaming。
5. holdout 端到端：只运行 current auto + current hybrid、最佳 ASR stem + current hybrid、最佳 ASR stem + overlap-only 辅助和已晋级 diar stem。

Demucs 路线优先复用隔离的 audio-separator==0.44.5 环境，从固定模型列表解析唯一的 htdemucs_ft 文件名；结果记录模型 ID、实际文件名和 SHA256。所有 stem 都转换为 16 kHz mono WAV，并检查非空、削波和相对原音频不超过 0.050 秒的时长漂移。SortFormer 使用 nvidia/diar_sortformer_4spk-v1（短音频 offline）和 nvidia/diar_streaming_sortformer_4spk-v2.1（完整节目 streaming），超过四位说话人时明确跳过。

基准输出包括 benchmark_manifest.json、benchmark_results.jsonl、benchmark_summary.csv、各 variant 的 ASS/RTTM/诊断和可下载 benchmark_bundle.zip。失败或 fallback variant 不参与均值，且不能冒充晋级成绩。结论只能是 reject、needs_more_data 或 promote_experimental，不会自动修改普通默认。

人工 ASS 的 Name 作为稳定 speaker ID；STAFF 默认忽略；重叠发言必须是独立 Dialogue。基准辅助逻辑与 ass-compare-diagnostics 的匹配、coverage、mismatch window 和短间隙语义保持一致。

## 使用与验证边界

1. 在 Colab 启用 GPU，按 Notebook 顺序运行初始化、文件选择、普通参数和主流程单元。
2. 运行模型加载、转写、Qwen 对齐、diarization、finalize/export，然后下载 SRT/ASS。
3. 高级基准只在私有样本 manifest 完整且需要时运行；不要把私有媒体、ASS、RTTM 或生成输出提交到仓库。
4. 生成成功不等于字幕正确：检查首尾覆盖、重复文本、speaker 切换、重叠 Dialogue 和 subtitle_gap_diagnostics.json。

本地 JSON/AST/纯函数测试不能证明真实 Colab GPU、Demucs、pyannote、NeMo、SortFormer 或私有样本流水线成功。模型相关改动必须按 docs/runbooks/notebook-validation.md 执行新鲜 T4 验证。

## 开发文档

- docs/architecture.md：普通流水线、输入边界、fallback 与基准产物。
- docs/decisions/ADR-0001-colab-notebook-delivery.md：单 Notebook 交付决策。
- docs/decisions/ADR-0002-speaker-overlap-backends.md：说话人后端、Demucs 和 SortFormer 决策。
- docs/runbooks/notebook-validation.md：静态、本地行为和新鲜 Colab 验证矩阵。
