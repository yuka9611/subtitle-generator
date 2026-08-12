# subtitle-generator

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/yuka9611/subtitle-generator/blob/main/subtitle_generator.ipynb)

使用 MOSS 生成带说话人和时间戳的 SRT/ASS 字幕。

## 使用

1. 在 Colab 中选择 GPU 运行时。
2. 按顺序运行 Notebook。
3. 选择音频或视频。
4. 设置语言、热词和预期说话人数。
5. 下载字幕或诊断包。

预期说话人数只用于诊断，不限制实际人数。ClearVoice 默认关闭。

详细架构和验证要求见 [docs/architecture.md](docs/architecture.md) 与 [docs/runbooks/notebook-validation.md](docs/runbooks/notebook-validation.md)。
