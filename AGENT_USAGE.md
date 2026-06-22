# LangChain / LangGraph PANDA-PUMA Agent 使用說明

目標寫一個 Python Script，使用者要表達他要做的任務，LLM 決定要用哪個工具或什麼都不執行。

## 這個 agent 在做什麼？

它不是取代 PANDA/PUMA，而是站在使用者和工具中間：

1. 使用者用自然語言說：「我要跑 PANDA」、「我要跑 PUMA」、「幫我看 input 格式」。
2. LLM 透過 OpenRouter 讀懂任務。
3. LangGraph 決定要不要呼叫工具。
4. 工具可以：
   - 解釋 PANDA/PUMA input-output
   - 檢查 input 檔案的基因/TF/miRNA ID 是否有交集
   - 執行 PANDA
   - 執行 PUMA
   - 如果資訊不足，就不執行，改問使用者補資料

## 為什麼要用 LangGraph？

一般 Python script 是固定流程：

```text
讀參數 -> 跑 PANDA
```

LangGraph 則適合做「LLM 先判斷，再決定要不要呼叫工具」的流程：

```text
使用者任務
  -> LLM 判斷
  -> 如果需要工具，呼叫工具
  -> 工具結果回到 LLM
  -> LLM 給使用者最後回覆
```

## 先設定 OpenRouter key

到 OpenRouter 建立 API key 後，在 shell 設定：

```bash
export OPENROUTER_API_KEY="sk-or-v1-..."
export OPENROUTER_MODEL="openai/gpt-4o-mini"
```

也可以參考 [.env.example](.env.example)。

## 在 Docker 裡使用

先 build：

```bash
docker compose build
```

互動模式：

```bash
docker compose run --rm \
  -e OPENROUTER_API_KEY="$OPENROUTER_API_KEY" \
  -e OPENROUTER_MODEL="$OPENROUTER_MODEL" \
  netzoo python scripts/netzoo_agent.py
```

直接給任務，但先不真的執行工具：

```bash
docker compose run --rm \
  -e OPENROUTER_API_KEY="$OPENROUTER_API_KEY" \
  netzoo python scripts/netzoo_agent.py \
  --task "我要用 data/expression.tsv data/motif.tsv data/ppi.tsv 跑 PANDA，輸出到 outputs/panda.tsv"
```

真的執行工具要加 `--execute`：

```bash
docker compose run --rm \
  -e OPENROUTER_API_KEY="$OPENROUTER_API_KEY" \
  netzoo python scripts/netzoo_agent.py \
  --execute \
  --task "我要用 data/expression.tsv data/motif.tsv data/ppi.tsv 跑 PANDA，輸出到 outputs/panda.tsv"
```

PUMA 範例：

```bash
docker compose run --rm \
  -e OPENROUTER_API_KEY="$OPENROUTER_API_KEY" \
  netzoo python scripts/netzoo_agent.py \
  --execute \
  --task "我要跑 PUMA，expression 是 data/expression.tsv，motif 是 data/motif.tsv，PPI 是 data/ppi.tsv，miRNA 是 data/mir.tsv，輸出 outputs/puma.tsv"
```

## 安全設計

預設不執行工具，只回傳它會執行的 command。這是為了避免 LLM 誤解任務時直接跑很久或覆蓋 output。

要真正執行才加：

```bash
--execute
```

## 跟 expression data preprocessing 的關係

expression data 要經過處理才能拿到基因表現 ：

```text
raw FASTQ
  -> QC
  -> trimming
  -> alignment / pseudoalignment
  -> quantification
  -> gene ID mapping
  -> filtering
  -> normalization
  -> expression matrix
  -> PANDA/PUMA
```

PANDA/PUMA 不吃原始 FASTQ。它們吃的是已經整理好的 gene expression matrix。
