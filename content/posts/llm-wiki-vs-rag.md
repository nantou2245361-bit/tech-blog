---
title: "LLM Wiki 與 RAG：知識要整理成文件，還是切塊檢索？"
date: 2026-11-17
tags: ["AI", "RAG", "知識管理"]
author: "阿翔"
---

你應該很熟這種場景：廠區 SOP 年年改版，客服看的是舊版 FAQ，RD 手上又有一份最新版規格，結果新人一來還是一直問同一題。你真正想解的，不是「模型會不會回答」，而是「知識到底要先整理成文件，還是切塊交給檢索」。這題如果只用 AI 熱潮來看，很容易吵成立場戰；但如果把它放回企業 IT 的日常，答案其實是條件式的：有些部門該先做 wiki，有些部門該先做 RAG，多數情況兩者會一起出現。

先把名詞講清楚。**LLM Wiki 不是官方或學術術語**；查核下來，DeepWiki、llms.txt、Anthropic 的 context engineering / Agent Skills 都是在做相近的事，但沒有把「LLM Wiki」當正式術語。可追溯的命名來源，是 Karpathy 在 2026-04-02 的 X 貼文與 2026-04-04 的 gist《LLM Wiki》；相對地，RAG 則是 Lewis 等人在 NeurIPS 2020 正式定義的術語 [1][2][3][4][6][7][8][9][10]。對主管來說，先分清楚名詞很重要，因為你要買的不是流行詞，而是知識治理的方法。

如果把機制攤開，兩者差很多：RAG 的核心是把文件切塊、向量化，查詢時再取回 top-k 片段；LLM Wiki 型做法則是把原始資料編譯成可讀、可維護的頁面，讓模型先看整頁或先靠 index 導航，再進一步回答。也就是說，RAG 比較像「查詢時找答案」，wiki 比較像「寫入時先把知識整理好」。這個差異很關鍵，因為它直接決定更新成本、檢索品質、成本結構與稽核方式 [1][6][8][9][10]。

| 面向 | RAG | LLM Wiki 型 |
|---|---|---|
| 知識單位 | chunk | 頁面 |
| 更新時機 | 查詢前維護索引 | ingest 時重寫受影響頁面 |
| 優勢 | 局部更新快 | 脈絡完整、可讀 |
| 失敗模式 | 片段斷章、檢索落空 | 編譯掉細節、頁面過時 |

**知識更新成本** 這一項，決定誰來背日常維護。RAG 的更新是索引維護：文件改了就重切、重算 embedding，局部自動化程度高，適合高頻異動的資料；但它也有新舊版本並存的風險，舊 chunk 沒刪乾淨時，檢索可能同時撈到兩版內容。LLM Wiki 的更新是內容維護：LLM 可以一次幫你改多頁、補交叉連結，但成本轉移到編譯與校對，還要定期 lint，抓矛盾、孤兒頁與過時主張。對主管來說，這一段的意義是：如果你的知識變動快到像工單、庫存、報價，那就別期待 wiki 追上速度；如果你的知識是 SOP、政策、架構與產品規格演進，wiki 才有價值 [1][11]。

**檢索品質** 才是這題最容易吵起來的地方。需要精確片段時，RAG 明顯占優：Anthropic 的 contextual retrieval 指出，僅靠原始 chunk 時，top-20 檢索失敗率是 5.7%，加上 contextual embeddings 降到 3.7%，再加 contextual BM25 降到 2.9%，最後再加 reranker 可到 1.9%；但這份實驗的指標是 1-recall@20、k=20、embedding 用 Gemini Text 004，而且原文沒有揭露確切 chunk size [11]。這對主管重要，因為法規逐條查詢、料號、錯誤碼、合約條號這類問題，本質上就是在比誰能抓到精準片段。

但當問題要跨章節串起來，整份文件或 wiki 的脈絡反而更穩。Lost in the Middle 顯示，在 20 篇文件、約 4K tokens 的條件下，GPT-3.5-Turbo 把答案放在第 1 位時準確率有 75.9%，移到第 10 位掉到 54.0%，甚至低於 closed-book 的 56.2% [12]；Google DeepMind 在 9 個資料集的對照也發現，資源充足（不計成本）時，長上下文平均勝過 RAG，Gemini-1.5-Pro、GPT-4o、GPT-3.5-Turbo 分別高出 7.6、13.1、3.6 個百分點，而且兩種方法對 63% 的查詢預測完全相同 [13]。如果問法一換，差距更明顯：DeepMind 的 PassKey 實驗裡，RAG 在原句下有 80.34%，但把問法改成「文中藏的特殊 token 是什麼」，RAG 直接掉到 4.58%，長上下文仍有 69.32% [13]。這一段告訴主管的是：當你要的是跨文件推理、前後依賴、或要讀完整脈絡，切塊檢索不一定夠。

**Token 成本與延遲** 不能只看直覺。Anthropic 官方價目表顯示，Claude Sonnet 5.5 輸入是每百萬 tokens 2 美元，prompt cache 讀取是 0.10 美元；Haiku 5.5 在提示超過 100K tokens 後，單價還會從 0.10 美元跳到 0.50 美元，也就是 5 倍 [15]。以同一份官方價目表估算，一個直白的例子：100K tokens 的整份 wiki 進 context，若不命中快取，每次約 0.20 美元；若命中快取約 0.01 美元；而 20 個、每個 800 tokens（假設值）的 RAG 片段約 16K tokens，約 0.032 美元。這裡的重點不是「wiki 一定比較貴」，而是它對快取命中率非常敏感。對每天上千次問答的客服或內部服務台，成本會先爆掉；對單題價值高、查詢量不大的投標、稽核、法務案，直接把相關文件整包放進上下文反而更省事。Anthropic 也明講：小於 200,000 tokens 的知識庫，其實可以直接整包進 prompt，不一定要先上 RAG [11]。

**可解釋性與可稽核性**，常常比模型分數更重要。RAG 的稽核可以細到 chunk，你可以回指原檔與段落；wiki 的稽核則以版本為單位，你可以看 diff、看誰改了什麼、看交叉連結怎麼變。這對 ISO、內控、交接特別有用 [1][3][4]。但企業真正要注意的是權限：Glean 官方強調的是 permissions-enforced enterprise search；如果你把整份 wiki 直接灌進 context，卻沒有在組裝層先處理權限，那模型看到的內容可能超出使用者應有範圍 [16]。所以這裡的重點不是「哪個比較會回答」，而是「哪個比較容易被稽核、比較不會把不該看的內容送進去」。

**規模上限** 會逼你做架構選擇。Anthropic 的官方說法很直接：小於 200K tokens 的知識庫，可以直接整包進 prompt；Karpathy 自己也提過，他的 wiki 大約有 100 篇文章、約 400K 字，靠的是 index 與工具讀取，不是整包灌進去 [1][2][11]。另外，Mintlify 的 llms.txt 實作還有 100,000 字元上限，超過就得拆成子索引 [7]。這些訊號合在一起，意思很明白：數百頁以內，整包或純 wiki 導航都還能做；頁數一多，你就需要搜尋層與治理流程；到了更大的規模，wiki 只能當地圖，不該硬扛所有查詢。

如果你想把兩者搭起來，最實際的是這種分層：原始文件是唯一事實來源，wiki 是骨架，RAG 是查詢層。新來源進來後，先產生摘要頁與索引，再更新受影響的 entity / concept 頁，接著跑 lint，最後經過 PR 審查再部署 [1][3][4][8][9]。觸發條件也可以很清楚：當查詢要的是「綜述＋精確事實」並存、當知識量已經超過整包上限、當你需要權限控管與引用時，混合架構最划算；當查詢幾乎全是單點事實，就讓 RAG 或資料庫處理；當查詢幾乎全是綜述，wiki 或全文上下文會更順手。

切塊檢索會贏的情境，很像企業裡的標準查詢：大量同質文件、每題只需要一段證據、而且要能精準引用某個段落。整份文件會贏的情境，則像例外處理：問題不是找單句，而是要把定義、前提、限制、變更歷程一起串起來。這不是模型偏好，而是資訊結構本來就不一樣。對主管來說，這句話的重點是：別先問「哪個模型比較強」，先問「你的知識長什麼樣子」。

混合架構真正的難點不在模型，而在治理。wiki 的 owner 不能只會寫頁面，還要知道哪些內容可以被模型摘要、哪些必須保留原文；RAG 的 owner 不能只會調向量庫，還要處理權限、版本淘汰與 prompt cache 命中率。只要有人把 wiki 當成第二份真相，最後一定會失控。比較穩的做法，是把原始文件設成唯一事實來源，把 wiki 當可讀地圖，把 RAG 當查詢加速器，三者各守一層。

給主管的選擇其實不難：如果你的部門主要管 SOP、政策、架構、制度，知識變動不算快，但大家一直問同一批「該怎麼做」的問題，先做 wiki 骨架會比較有感；如果你的知識高頻異動，而且答案多半是單點事實，先做 RAG 或直接接資料庫比較合理；如果你有嚴格權限、又要稽核軌跡，那就別選邊站，做混合架構。真正成熟的做法，不是把文件和檢索對立起來，而是讓 wiki 負責整理脈絡，讓 RAG 負責在需要時把那一段精準抓出來 [11][13][16]。

如果你還在做投資評估，先問三個數字：知識庫大約多少 tokens、查詢裡單點事實與綜述各占多少、誰來負責 lint。這三題答不出來，就先別急著選模型，因為你連維運模式都還沒定。若沒有 lint 與 PR，wiki 很快就會變成一堆漂亮但過期的說明頁；那時候你失去的不是文件，而是大家對答案的信任。模型可以幫你整理，但不能替你決定哪一頁是權威版本。這比換模型更重要，也更難外包。

## 參考文獻

1. Andrej Karpathy, *LLM Wiki*（gist）— https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
2. Andrej Karpathy, "LLM Knowledge Bases"（X 貼文）— https://x.com/karpathy/status/2039805659525644595
3. Cognition, *DeepWiki: AI docs for any repo* — https://cognition.com/blog/deepwiki
4. Cognition／Devin Docs, *DeepWiki repository wikis* — https://docs.devin.ai/work-with-devin/deepwiki
5. DeepWiki 首頁 — https://deepwiki.com/
6. llms.txt 規格（v2）— https://llmstxt.org/
7. Mintlify Docs, *llms.txt* — https://www.mintlify.com/docs/ai/llmstxt
8. Anthropic, *Effective context engineering for AI agents* — https://www.anthropic.com/research/effective-context-engineering-for-ai-agents
9. Anthropic, *Introducing Agent Skills* — https://claude.com/resources/articles/skills
10. Lewis et al., *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks* — https://arxiv.org/abs/2005.11401
11. Anthropic, *Introducing Contextual Retrieval* — https://www.anthropic.com/research/contextual-retrieval
12. Liu et al., *Lost in the Middle: How Language Models Use Long Contexts* — https://arxiv.org/abs/2307.03172
13. Li et al. (Google DeepMind), *Retrieval Augmented Generation or Long-Context LLMs? A Comprehensive Study and Hybrid Approach* — https://arxiv.org/abs/2407.16833
14. Hong, Troynikov, Huber (Chroma), *Context Rot: How Increasing Input Tokens Impacts LLM Performance* — https://www.trychroma.com/research/context-rot
15. Anthropic, *Plans & pricing — API*（抓取日 2026-11-17）— https://claude.com/pricing#api
16. Glean, *Glean Assistant* — https://www.glean.com/ai-assistant
