# 分词器学习笔记

## 引言
本文档基于与AI助手Kimi的对话，整理了关于分词器（Tokenizer）的学习内容，特别是GPT系列模型中使用的字节级BPE分词器。内容从基础概念到实际代码实现，帮助初学者理解分词器的原理和应用。

## 1. 分词器基础概念

### 什么是分词器
- 计算机只能处理数字，因此需要将文本转换为数字序列。
- 最原始的分词器：按字符切分，每个字符对应一个数字（例如：H→72, e→101）。
- 问题：序列过长，缺乏语义信息。
- 现代分词器：使用子词（subword）切分，既能覆盖常见片段，又能拆分罕见词。

### 核心思想
“把文本切成‘子词’片段，用尽可能少的token数，同时保留把罕见词拆成常见片段的能力。”

### 技术路线
1. **BPE（Byte-Pair Encoding）**：从字符起步，每次合并最常相邻出现的两个小块。
2. **WordPiece**：类似BPE，但合并策略用似然增益而非频率最高。
3. **Unigram LM**：先候选子词，再砍掉概率最低的。

GPT家族使用BPE变种。

## 2. GPT训练时使用的分词器

### 版本演进
- **GPT-1**：BPE，词汇表40,000个条目。
- **GPT-2**：BPE，开源实现gpt2-tokenizer，词汇表50,257个条目。
  - 关键：先转UTF-8字节流，再对字节做BPE，确保任何Unicode文本都能被编码，无OOV。
- **GPT-3/ChatGPT/GPT-4**：沿用GPT-2方案，词汇表约50k，开源实现tiktoken。

### 字节级BPE流程示例
句子：“我爱吃🍎”
1. UTF-8字节化：我→e6 88 91, 爱→e7 88 b1, 吃→e5 90 83, 🍎→f0 9f 8d 8e
2. BPE合并：根据训练规则合并高频字节对。
3. 输出整数序列：[36423, 28892, 34522, 127849]

## 3. 与中文分词的区别
- 传统中文分词：语义级切词，如“我爱吃苹果”→[我/爱/吃/苹果]。
- GPT的BPE：不care语义，只看字节共现频率。
- 示例：
  - “苹果”可能整体token。
  - “枇杷”可能拆成两个token。
  - “人工智能”可能切成“人工”+“智能”。

## 4. 训练分词器的必要性
- 分词器规则由训练数据统计得出。
- 不同领域数据有不同高频片段：
  - 新闻：“疫情”高频。
  - 二次元：“喵帕斯”高频。
  - 中医：“黄芩”高频。
- 使用不匹配词表会导致序列变长、语义丢失。

### 为什么训练数据要作为分词器的输入，而不用传统分词？
- **BPE的本质**：BPE不是预定义的固定词典或规则，而是通过统计训练数据中相邻字节对（或子词对）的共现频率来动态学习合并规则。每一步合并都是基于“当前语料中最常出现的相邻片段”，这使得分词器完全由数据驱动，而不是依赖人工设计的词典。
- **传统分词的局限性**：传统分词（如中文分词）依赖语言专家知识和预构建词典，旨在进行语义级切词（如将句子切成有意义的词语），适用于特定任务（如命名实体识别或信息检索）。然而，在通用语言模型（如GPT）中，传统分词无法处理多语言混合、罕见词、emoji或代码片段，且词典更新困难，无法适应新领域的数据分布。
- **数据驱动的优势**：将训练数据作为输入，确保分词器学习到的合并规则与模型即将训练的数据分布一致。这能优化token效率（减少序列长度，提高训练速度和内存利用），并保留子词级别的语义信息。例如，在对话数据中，“用户:”和“助手:”这样的高频模板会被合并成单个token，避免浪费上下文长度。
- **对比总结**：传统分词是静态的、语义导向的；BPE是动态的、统计导向的。通过用自己的训练数据训练分词器，我们能获得一个“量身定制”的token表，而非通用的、不匹配的词表，从而提升模型的整体性能。

### 训练代码示例
```python
from tokenizers import Tokenizer, models, pre_tokenizers, decoders, trainers, processors

def build(train_path: str, save_path: str, vocab_size: int = 2048):
    tokenizer = Tokenizer(models.BPE(unk_token="<unk>"))
    tokenizer.pre_tokenizer = pre_tokenizers.ByteLevel(add_prefix_space=False)
    tokenizer.decoder = decoders.ByteLevel()
    trainer = trainers.BpeTrainer(vocab_size=vocab_size, special_tokens=["<bos>", "<eos>", "<pad>", "<unk>"])
    texts = []
    with open(train_path, "r", encoding="utf-8") as f:
        for line in f:
            obj = json.loads(line)
            prompt = obj.get("prompt", "")
            completion = obj.get("completion", "")
            texts.append("用户:" + prompt + "\n助手:" + completion)
    tokenizer.train_from_iterator(texts, trainer)
    tokenizer.post_processor = processors.TemplateProcessing(
        single="<bos> $A <eos>",
        pair=None,
        special_tokens=[("<bos>", tokenizer.token_to_id("<bos>")), ("<eos>", tokenizer.token_to_id("<eos>"))],
    )
    tokenizer.save(save_path)
```

## 5. 词表“乱码”问题分析

### 现象
词表中出现如"´æĺİ": 1110的条目，看起来像乱码。

### 根因
- tokenizer底层存储原始bytes。
- 为便于json序列化，先用latin-1解码成str。
- latin-1只能表示0-255，非法映射生成\u010a等转义字符。
- 再次encode('latin-1')时出错。

### 解决方法
使用tokenizer.model.tokens直接获取原始bytes，避免latin-1陷阱。

### 安全查看词表代码
```python
def print_vocab(tokenizer, top_k=50):
    tokens = tokenizer.model.tokens
    for idx in range(min(top_k, len(tokens))):
        raw = tokens[idx]
        try:
            readable = raw.decode("utf-8", errors="backslashreplace")
        except Exception:
            readable = repr(raw)
        if any(c in readable for c in "\n\t\r "):
            readable = repr(readable)
        print(f"{idx:5d}  {readable}")
```

## 6. 动手实践
### 安装和使用tiktoken
```python
import tiktoken
enc = tiktoken.get_encoding("gpt2")
ids = enc.encode("我爱吃🍎")
print(ids)  # [36423, 28892, 34522, 127849]
print(enc.decode(ids))  # 我爱吃🍎
```

### 训练和查看脚本
参考上述代码，训练后使用print_vocab查看词表。

## 7. 总结
1. 分词器是文本到整数列表的翻译官，效率影响模型性能。
2. GPT使用字节级BPE，vocab~50k，tiktoken是开源实现。
3. 先UTF-8字节化再BPE，确保无OOV，但可能拆分汉字。
4. 词表由数据统计，需用自己的数据训练。
5. “乱码”是显示问题，用原始bytes避免。

## 常见疑问
- 英文单词拆分：罕见词被拆成常见片段。
- 中文拆分：可能2-3个token，生僻字常见。
- 训练中文GPT：先用tiktoken，再扩充高频词。

通过这些内容，希望能帮助理解分词器的原理和实践。