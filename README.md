# subtitle-generator

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/yuka9611/subtitle-generator/blob/main/subtitle_generator.ipynb)

使用 MOSS 生成带说话人和时间戳的 SRT/ASS 字幕。

## 使用

1. 在 Colab 中选择 GPU 运行时。
2. 配置 `HF_TOKEN`（pyannote 全局说话人识别需要）：先打开 https://huggingface.co/pyannote/speaker-diarization-community-1 点击 **Agree and access repository** 接受许可，再在 https://huggingface.co/settings/tokens 创建 **Fine-grained** Access Token，勾选 **Read access** 与 **Access to public gated repositories / gated models**，最后在 Colab 右侧 **🔑 Secrets** 新建 `HF_TOKEN` 并打开本 Notebook 的访问开关。
3. 按顺序运行 Notebook（未配置 `HF_TOKEN` 时会自动回退到 MOSS 拼接 + 后处理合并，不会中断）。
4. 选择音频或视频。
5. 设置语言、热词和预期说话人数。
6. 下载与视频同名的 SRT/ASS 或诊断包。

预期说话人数：留空为自动；整数 1–99 会把最终说话人固定为该数量；`8+` 表示至少 8 人。人数由 pyannote community-1 在整段音频上全局约束（`num_speakers` / `min_speakers=8` / 自动），再把 MOSS 各分块的 local speaker 按时间 overlap 映射到全局 `S01…S0N`；检测到的人数不足目标时不虚构，保留实际人数并记录警告。pyannote 不可用时回退到证据拼接 + 确定性合并。热词预设（原神/星铁）已扩充为游戏内专有名词（概念、地名、人名及日语声优名）。ClearVoice 默认关闭。

详细架构和验证要求见 [docs/architecture.md](docs/architecture.md) 与 [docs/runbooks/notebook-validation.md](docs/runbooks/notebook-validation.md)。
