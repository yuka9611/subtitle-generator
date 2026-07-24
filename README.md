# subtitle-generator

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/yuka9611/subtitle-generator/blob/main/subtitle_generator.ipynb)

Google Colab 字幕生成 Notebook，面向中文及混合语言音视频。主流程使用 faster-whisper/Qwen ASR、Qwen Forced Align 和可选 NeMo diarization，最终导出 SRT、ASS 及字幕间隙诊断。

## 流程

```text
音频/视频
-> FFmpeg 预处理
-> faster-whisper 主识别 + Qwen 局部/尾部补救
-> Qwen 词级强制对齐
-> NeMo 说话人分离（可选）
-> 断句、重叠轨与短反应处理
-> SRT / ASS / subtitle_gap_diagnostics.json
```

## 使用

1. 点击上方 Colab 按钮。
2. 依次运行 Notebook 单元，不要跳过环境初始化和运行参数。
3. 上传媒体并选择识别、语言、说话人数和字幕样式。
4. 运行识别、对齐及需要的 diarization 单元。
5. 运行导出单元并下载 SRT/ASS。

Notebook 会安装并固定一组对 NumPy、Transformers、Qwen ASR、faster-whisper 和 NeMo 兼容的依赖。调整版本时需要在真实 Colab GPU 运行时重新验证完整流程。

## 输出检查

生成成功不等于字幕正确。至少检查：

- 开头与结尾是否完整，是否有明显内部空洞。
- 时间戳、断句、短反应和说话人切换。
- 多人重叠时 ASS 主/副轨、层级和样式。
- SRT/ASS 中文编码和播放器兼容性。
- `subtitle_gap_diagnostics.json` 中的高风险区间。

对照人工修订 ASS 时可使用 `$ass-compare-diagnostics`。

## 开发

项目的执行源文件是 `subtitle_generator.ipynb`。修改时保持 cell 顺序、Colab 表单元数据和依赖约束，避免整份 Notebook 无关重写。

最小静态检查：

```bash
python -m json.tool subtitle_generator.ipynb >/dev/null
git diff -- subtitle_generator.ipynb
```

涉及模型、CUDA、依赖、对齐、diarization 或导出的变更必须按 `docs/runbooks/notebook-validation.md` 在 Colab 和真实样本上验证。

## 文档

- `docs/architecture.md`：Notebook 流水线与失败边界。
- `docs/decisions/ADR-0001-colab-notebook-delivery.md`：单 Notebook 交付决策。
- `docs/runbooks/notebook-validation.md`：修改和验收矩阵。
