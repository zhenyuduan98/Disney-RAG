# Disney-RAG — 迪士尼客服多模态 RAG 助手

本项目是一个面向迪士尼客服场景的 **多模态 RAG（检索增强生成）助手**：它能解析 Word 文档（文本 + 表格）和图片（OCR + 视觉特征），把内容向量化后存入 FAISS 向量库；当用户提问时，先从知识库中检索相关内容（文本 + 图片），再交给大模型生成「有依据、不瞎编」的客服回答。

项目同时配套演示了 RAG 工程里最关键的一环——**文本切片（Chunking）策略**，提供固定长度、语义、LLM 语义、层次、滑动窗口共 5 种切片方式的对比实现，以及基于 Qwen-VL 的图像理解示例。

---

## 核心功能

- **多格式文档解析**：从 Word 文档中提取段落文本与表格（表格转为 Markdown），保留结构信息。
- **图文多模态**：
  - 文本向量：`text-embedding-v4`（阿里云百炼 / DashScope）
  - 图片向量：CLIP（`openai/clip-vit-base-patch32`）
  - 图片文字：Tesseract OCR（中英文）
- **向量检索**：使用 FAISS 分别为文本与图片建立索引；提问时向量化查询并做相似度检索，命中图片关键词（如「海报」「万圣节」）时自动触发图像检索。
- **受约束生成**：把检索到的「背景知识」拼进 Prompt，要求模型 `qwen-plus` 仅依据背景知识作答，降低幻觉。
- **切片策略对比**：6 个独立脚本演示不同 Chunking 方式对检索效果的影响，便于教学与调参。

---

## RAG 流程

```
知识库文档（Word / 图片）
        │  解析：python-docx 提取文本+表格；Tesseract OCR + CLIP 处理图片
        ▼
   文本切片 / 向量化（text-embedding-v4、CLIP）
        │
        ▼
   FAISS 向量索引（文本索引 + 图片索引）
        │
   用户提问 ──► 向量检索（Top-k 文本，必要时检索图片）
        │
        ▼
   构建 Prompt（背景知识 + 问题）──► qwen-plus 生成答案
```

---

## 目录结构

```
Disney-RAG/
├── README.md
├── 1-disney_bot.py / .ipynb          # 主程序：多模态 RAG 客服助手完整流程
├── 1-固定长度切片.py / .ipynb         # 切片策略：固定长度
├── 2-语义切片.py / .ipynb             # 切片策略：基于语义
├── 3-LLM语义切片.py / .ipynb          # 切片策略：用 LLM 做语义切分
├── 4-层次切片.py / .ipynb             # 切片策略：层次化
├── 5-滑动窗口切片.py / .ipynb          # 切片策略：滑动窗口
├── 6-Qwen-VL图像理解.py / .ipynb       # 多模态：Qwen-VL 图像理解
├── 2-万圣节.jpeg                       # 示例图片
├── tesseract-ocr-w64-setup-*.exe       # Tesseract OCR 安装包（Windows，便于本地配置）
├── disney_knowledge_base/              # 主程序实际使用的知识库
│   ├── 1-上海迪士尼门票规则.docx 等文档
│   └── images/                         # 图片（万圣节海报等）
└── 迪士尼RAG知识库/                     # 完整知识库（分 5 大类，供扩展检索）
    ├── 1-产品与服务详情/
    ├── 2-运营流程与标准作业程序/
    ├── 3-特殊情况与应急预案/
    ├── 4-客户关系与支持话术/
    └── 5-内部知识与工具/
```

> 主程序 `1-disney_bot.py` 默认读取 `disney_knowledge_base/` 作为知识库；`迪士尼RAG知识库/` 是更完整的分类资料库，可按需替换或扩展检索范围。

---

## 快速开始

### 1. 安装 Python 依赖

```bash
pip install openai "faiss-cpu" python-docx PyMuPDF Pillow pytesseract transformers torch requests
```

### 2. 安装 Tesseract OCR 引擎

图片 OCR 依赖 Google Tesseract：

- **Windows**：运行仓库内的 `tesseract-ocr-w64-setup-*.exe`，或从 [UB-Mannheim/tesseract](https://github.com/UB-Mannheim/tesseract/wiki) 下载安装；安装时勾选中文语言包（`chi_sim`）。
- **macOS**：`brew install tesseract tesseract-lang`
- **Linux**：`sudo apt-get install tesseract-ocr tesseract-ocr-chi-sim`

请确保 `tesseract` 可执行文件已加入系统 PATH。

### 3. 配置环境变量

```bash
# 阿里云百炼 API Key（调用 text-embedding-v4 / qwen-plus / Qwen-VL）
export DASHSCOPE_API_KEY="你的_API_Key"        # Windows PowerShell: $env:DASHSCOPE_API_KEY="..."

# 可选：Hugging Face Token，用于下载 CLIP 模型
export HF_TOKEN="你的_HF_Token"
```

### 4. 运行

```bash
python 1-disney_bot.py
```

程序会先离线构建知识库索引，然后运行内置的示例问答（门票退款流程、万圣节活动海报、年卡优惠等）。把 `rag_ask(query=...)` 换成你自己的问题即可。

---

## 技术栈

- **大模型 / Embedding**：通义千问 `qwen-plus`、`text-embedding-v4`、Qwen-VL（经 DashScope 调用）
- **多模态**：CLIP（`transformers`）、Tesseract OCR（`pytesseract`）
- **向量检索**：FAISS（`faiss-cpu`）
- **文档解析**：`python-docx`、PyMuPDF（`fitz`）、Pillow
- **运行环境**：Python、Jupyter Notebook

---


