---
layout: post
title: "用 Qwen3-4B 微調搜尋擴展：提升 KOL 搜尋匹配度"
date: 2026-03-05
categories: [AI/ML 技術]
tags: [Machine Learning, NLP, AI, Deep Learning]
author: Dennies Hong
---

使用者搜「台灣美妝 YouTuber」，搜尋引擎只會找完全匹配的文字。

但 KOL 的 profile 裡可能寫的是「beauty content creator」或「彩妝教學頻道」——跟使用者打的字一個都對不上，但明明就是要找的人。KOL 搜尋場景這個問題特別嚴重，因為每個 KOL 的自我描述都不一樣，沒有統一格式。

解決方案是 Query Expansion — 把使用者的搜尋查詢擴展成多種形式，讓搜尋引擎能覆蓋更多匹配面向。直接用 GPT-4 來做效果不錯但太慢太貴。我們的做法是用大模型生成訓練資料，再 fine-tune 一個 Qwen3-4B 的小模型來跑 query expansion，速度快、成本低、可以本地部署。

## 三條線：lex、vec、hyde

Query expansion 不是把搜尋詞翻譯成同義詞這麼簡單。我們設計了三種擴展，分別對應三種搜尋技術：


<div class="mermaid">
graph LR
    Q["原始查詢"]:::query --> LEX["lex<br/>關鍵字變體"]:::lex
    Q --> VEC["vec<br/>語意重述"]:::vec
    Q --> HYDE["hyde<br/>假文件生成"]:::hyde
    LEX --> BM25["BM25<br/>全文檢索"]:::search
    VEC --> VS["向量搜尋"]:::search
    HYDE --> VS
    BM25 --> RES["合併搜尋結果"]:::result
    VS --> RES
    style Q fill:#2c3e50,stroke:#1a252f,color:#fff
    style LEX fill:#e74c3c,stroke:#c0392b,color:#fff
    style VEC fill:#3498db,stroke:#2980b9,color:#fff
    style HYDE fill:#9b59b6,stroke:#8e44ad,color:#fff
    style BM25 fill:#f39c12,stroke:#e67e22,color:#fff
    style VS fill:#1abc9c,stroke:#16a085,color:#fff
    style RES fill:#2ecc71,stroke:#27ae60,color:#fff
</div>

**lex（Lexical）**：給 BM25 全文檢索用。把 query 擴展成一組關鍵字變體，包含中英文同義詞、縮寫、業界術語。

```
原始: "台灣美妝 YouTuber"
lex:  美妝 KOL YouTube 台灣 美妝網紅 化妝教學 創作者
```

BM25 比對的是字面文字，所以 lex 的目標是把使用者沒打出來但相關的詞都補上。「YouTuber」和「創作者」是同一個意思，「美妝」和「化妝教學」高度相關 — 這些都是使用者心裡有但沒打出來的詞。

**vec（Vector）**：給向量搜尋用。用自然語言重新描述同一個搜尋意圖。

```
vec: 在 YouTube 上活躍的台灣美妝 KOL
```

向量搜尋比對的是語意，換一種說法能找到不同的相關文件。一句短 query 的 embedding 有時不夠精確，用更完整的自然語言描述能產生更好的向量。

**hyde（Hypothetical Document Embedding）**：也給向量搜尋用，但思路完全不同。不改寫 query，而是直接「假裝寫一段能回答這個查詢的文件」。

```
hyde: 【YouTube創作者】台灣美妝KOL，粉絲數85萬，
      互動率6.1%，主要內容為化妝教學和產品評測...
```

為什麼這招有效？一句「台灣美妝 YouTuber」的 embedding 和資料庫裡真實的 KOL profile 之間有語意鴻溝 — query 太短，profile 太詳細。但如果我們先生成一段「長得像 KOL profile」的假文件，這段假文件的 embedding 會和真實 profile 在向量空間非常接近。用假文件的向量去搜，就能找到更精準的結果。

這三條線各自負責不同的搜尋面向：lex 確保關鍵字覆蓋、vec 確保語意匹配、hyde 確保和資料庫的文件格式對齊。

## Pipeline：五步走通


<div class="mermaid">
graph TD
    S1["Step 1: GPT-4.1-mini<br/>生成訓練資料"]:::s1 --> S2["Step 2: 品質驗證<br/>5 維度自動檢查"]:::s2
    S2 --> S3["Step 3: 格式轉換<br/>Chat Format"]:::s3
    S3 --> S4["Step 4: LoRA 微調<br/>Qwen3-4B"]:::s4
    S4 --> S5["Step 5: 測試推理<br/>Merged 模型部署"]:::s5
    style S1 fill:#e74c3c,stroke:#c0392b,color:#fff
    style S2 fill:#f39c12,stroke:#e67e22,color:#fff
    style S3 fill:#3498db,stroke:#2980b9,color:#fff
    style S4 fill:#9b59b6,stroke:#8e44ad,color:#fff
    style S5 fill:#2ecc71,stroke:#27ae60,color:#fff
</div>

整個 fine-tuning pipeline 分五步：

**Step 1：生成訓練資料**。用 Azure OpenAI 的 GPT-4.1-mini 批量生成。先準備一組種子查詢（從搜尋 log 和 KOL 資料庫抽取常見 pattern），然後讓 GPT 為每個查詢生成 lex/vec/hyde 三條線的擴展。還可以用 GPT 從種子查詢生成更多變體，快速擴充資料量。

**Step 2：驗證品質**。自動化檢查五個維度 — 格式是否完整（有沒有三條線都有）、lex 的關鍵字數量夠不夠、hyde 的長度是否足夠。品質不過關的資料直接剔除。

**Step 3：格式轉換**。把 JSONL 資料轉成模型訓練需要的 chat format。Qwen3 用的是 `<|im_start|>` / `<|im_end|>` 的格式，加上 `/no_think` 前綴告訴模型直接輸出結果不要進思考模式。

**Step 4：訓練模型**。用 LoRA 微調，rank=16、alpha=16、dropout=0.005。target modules 包括 attention 的 q/k/v/o_proj 和 FFN 的 gate/up/down_proj，加上 embed_tokens 和 lm_head 做完整訓練。3 個 epoch，learning rate 2e-4，cosine scheduler。一張 24GB GPU（RTX 4090 / A100 / L4）大約跑 10-30 分鐘。

**Step 5：測試推理**。訓練完成後自動產生兩個目錄 — LoRA adapter（幾十 MB）和 merged 完整模型（約 8GB）。部署用 merged 版本，直接載入不用先載基底模型再套 adapter。

## 資料量的 sweet spot


我們測了不同資料量的效果：

- 200-500 筆：學會格式，品質一般。最低門檻，先跑起來驗證 pipeline 可用
- 500-1000 筆：品質不錯，能實際使用。這是推薦的起步量
- 1000-2000 筆：品質好，覆蓋面廣
- 2000 筆以上：邊際效益遞減，除非 domain 非常多樣

我認為 500-1000 筆是最划算的區間。用 GPT-4.1-mini 生成 1000 筆訓練資料的成本大概幾美元，但訓練出來的 4B 模型在部署時每次 inference 的成本幾乎為零。

## 為什麼用 SFT 而不是 GRPO

這個專案的 fine-tuning guide 裡其實有提到兩階段訓練的可能性：先 SFT 學格式和基礎能力，再用 GRPO（強化學習）做品質優化。

但建議是先只做 SFT。原因很實際：SFT 已經能達到 80% 的效果，GRPO 是錦上添花。SFT 的訓練流程簡單（給範例、學著做），出問題容易 debug。GRPO 需要設計 reward function — 對 query expansion 這個任務來說，什麼是「好的擴展」並不容易量化。lex 的關鍵字覆蓋率可以自動評估，但 hyde 生成的假文件品質要怎麼打分？

等 SFT 版本上線跑一陣子，有了真實的搜尋品質數據，再回頭設計 reward function 做 GRPO，這個路徑更穩。

## 版本管理的實務

一個實務上很容易被忽略的問題：每次重新訓練時怎麼管理不同版本的模型？

做法是每次訓練用不同的 output 路徑和對應的 config 檔。v1 用 `configs/sft.yaml`，output 在 `sft/` 和 `sft-merged/`。v2 複製一份 config 改成 `configs/sft-v2.yaml`，output 改到 `sft-v2/` 和 `sft-v2-merged/`。每個 config 檔獨立保留，方便回溯「當時是用什麼設定訓練的」。

這種做法比起用 git branch 管理模型版本來得直覺 — config 檔裡寫著所有超參數，model output 路徑直接指向對應的 checkpoint。要比較兩個版本就 diff 兩份 config，然後分別用兩個 merged 模型跑 inference 比結果。

## LoRA 的設計考量

這個專案的 LoRA 配置有幾個值得注意的選擇：

rank=16 搭配 alpha=16（ratio=1），比常見的 alpha=2x rank 更保守。rank 16 對 4B 模型來說是中等偏大的配置，足夠讓模型學會 query expansion 的模式，但不會過度改變基底模型的知識。

target modules 涵蓋了 attention 和 FFN 的所有核心層（q/k/v/o_proj + gate/up/down_proj），但 embed_tokens 和 lm_head 用的是完整訓練而非 LoRA。這是因為 query expansion 需要模型學會新的輸出格式（lex/vec/hyde），embed_tokens 和 lm_head 直接影響模型對新 token pattern 的理解和生成能力，用 LoRA 可能不夠。

dropout 設得很低（0.005），因為訓練資料量本身不大（幾百到幾千筆），主要靠 LoRA 的低 rank 本身來做 regularization，額外的 dropout 不需要太強。

4-bit 量化（bitsandbytes）在 24GB GPU 上是必開的選項。4B 模型的 bf16 權重大約需要 8GB VRAM，加上 LoRA 的可訓練參數、optimizer states、gradient，24GB 會很緊。4-bit 量化把基底模型的 VRAM 壓到 2-3GB，剩下的空間全給訓練過程用。

---

Query expansion 本質上是「意圖放大」——把幾個字擴展成完整的搜尋意圖描述。但使用者的意圖有時候本身就是模糊的，搜「運動」的人可能自己都不確定要找健身教練還是運動品牌代言人。擴展能覆蓋更多可能性，但也會引入噪音。什麼時候該忠於原始 query、什麼時候該大膽擴展？這個平衡點大概只能靠 A/B test 去逼近，但 A/B test 的指標要怎麼設計也不簡單。

---

*Dennies Hong — AI/ML Product Manager / 寫技術、寫產品、寫真實經歷。*
