# subtitle-generator

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/yuka9611/subtitle-generator/blob/main/subtitle_generator.ipynb)

使用 MOSS 生成带说话人和时间戳的 SRT/ASS 字幕。

## 使用

1. 在 Colab 中选择 GPU 运行时。
2. 按顺序运行 Notebook。
3. 选择音频或视频。
4. 设置语言、热词和预期说话人数。
5. 下载与视频同名的 SRT/ASS 或诊断包。

预期说话人数：留空为自动；整数 1–99 会把最终说话人固定为该数量（检测到的人数不足时不虚构，保留实际人数并记录警告）；`8+` 表示至少 8 人。热词预设（原神/星铁）已扩充为游戏内专有名词（概念、地名、人名及日语声优名）。ClearVoice 默认关闭。

详细架构和验证要求见 [docs/architecture.md](docs/architecture.md) 与 [docs/runbooks/notebook-validation.md](docs/runbooks/notebook-validation.md)。
