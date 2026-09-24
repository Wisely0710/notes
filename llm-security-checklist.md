---
layout: default
title: "把 AI 安全清單從願望變成斷言：28 項、兩個公開 repo 的實測"
---

# 把 AI 安全清單從願望變成斷言：28 項、兩個公開 repo 的實測

> 作者：Wisely（<https://github.com/Wisely0710>）｜2026-09
> 文中的每一項判定都指向公開 repo 的檔案或可重跑指令： [`x402-agent-payments`](https://github.com/Wisely0710/x402-agent-payments) @ `8ca7727`、[`docs-rag`](https://github.com/Wisely0710/docs-rag) @ `5a76d9a`。
> **2026-09-24 更新**：文中指出的 7 處「宣稱 vs 實作」落差**已全數修補並推送**（`x402-agent-payments` `39aeffc`：metadata fail-closed＋文件對齊；`docs-rag` `b3168f4`：multi-tenant 用語、FTS-only 退路、日誌落點）；本文保留發布日的判定與敘述，第四、五節各補一則註記。

## 一、安全清單為什麼會腐化

「確保安全」、「遵循最小權限」、「注意 prompt injection」——這三句可以貼在任何產品的 README 上，而且永遠是真的，也永遠沒有用。

一份清單要能生效，每個條目必須滿足兩個條件：**能勾選**（有明確的是／否），以及**能驗收**（有一條斷言會因為它失效而變紅）。做不到這兩件事的條目，三個月後就會變成歷史文件。

所以我把清單設計成 7 組、28 項：

- **A. 能力與授權**——為什麼要這組：先確定「agent 碰得到什麼」是最小集合，後面每一組防線才有意義。工具清單逐項標註任務必需／可移除；工具參數有 JSON Schema 且驗證在程式碼；下游以最小權限身分存取；授權矩陣有測試斷言（未授權呼叫必須失敗）。
- **B. 不可逆操作與自主性**——不可逆動作（付款、寄信、刪除、公開發布、改權限）已列冊並有核准路徑；對外操作有速率與金額額度且超限排隊；對外連線預設封閉加白名單；執行環境沙箱化且實測生效。
- **C. 輸入、輸出與資料**——外部內容明確標記為資料；模型輸出進下游前必經 schema 驗證；不直接執行模型產生的程式碼；檢索階段即套用存取控制、快取鍵含租戶／權限／prompt 版本；受監管資料的路由由程式強制；log／trace／eval 分級設保存期限，原文存取要授權並留稽核。
- **D. 資源與成本**——迴圈有步數／token／金額硬上限且偵測重複呼叫；單次輸出長度與工具呼叫次數有上限；成本與用量有儀表與告警。
- **E. 監控與事故應變**——每一步都有可回放的 trace；異常工具呼叫與出網嘗試有告警規則與負責人；kill switch 存在且演練過；控制面的失敗姿態顯式設為 fail-closed 並驗證過。
- **F. 驗證與紅隊**——紅隊涵蓋直接／間接／跨模態注入；有惡意 MCP server 或惡意文件的可重現測試；注入與外洩案例納入回歸集；金絲雀權杖測試通過。
- **G. 治理與文件**——風險對映到 OWASP LLM Top 10 2026、Agentic Top 10 2026 與 MITRE ATLAS；清單本身是公開可讀文件；未實作項與其風險明文記錄。

## 二、判定只有四種，而且「已實作」只認碼與測試

| 判定 | 意思 |
|---|---|
| 已實作 | 有程式碼路徑，且有測試會因它失效而變紅 |
| 部分 | 有部分機制，或缺測試，或覆蓋面窄於字面要求 |
| 未實作 | 找不到實作或執行點 |
| 不適用 | 該 repo 不存在此風險面（必須附理由） |

規則是：**註解與 README 不算證據**。這一條在第 4 節直接抓出了 7 處落差。

## 三、實測（2026-09-20 快照）：56 個格子裡，只有 4 個是「已實作」

我對兩個自己寫的公開服務逐項判定：

- `x402-agent-payments`：agent 付費橋。verifier 是資源方（只驗 EIP-712 簽章、不持資金、不送交易），client 是瀏覽器端簽章，mcp 是買方工具。
- `docs-rag`：本地文件檢索服務（MCP 暴露 `retrieve`／`corpus_status`），外加一層 QA 評測。

| 判定 | x402-agent-payments | docs-rag | 合計 |
|---|---|---|---|
| 已實作 | 3 | 1 | 4 |
| 部分 | 8 | 15 | 23 |
| 未實作 | 11 | 9 | 20 |
| 不適用 | 6 | 3 | 9 |

**沒有任何一項在兩個 repo 都是「已實作」。** 4 個已實作項是 `x402` 的 A3／A4／E4（授權判定在資源方以真簽章完成、拒絕矩陣有測試、未知網路／條款不符／缺簽章一律顯式失敗）與 `docs-rag` 的 C3（模型輸出是純文字，不被執行、也不回饋進工具）。

（**2026-09-24 更新**：上表是 2026-09-20、以 `8ca7727`／`5a76d9a` 為基準的判定快照。其後兩批修補已使 4 格由「部分」升為「已實作」——`docs-rag` 的 C1／D3／E1／E4（2026-09-20 營運層硬化，見 `llm_security_audit.md` §九），合計由 4 升至 8；x402 的 fail-closed 落差亦已關閉（2026-09-24 `39aeffc`，見 §十）。逐項現值與可重跑證據以該文件為準。）

這個分佈本身就是結論：**我原本以為「有寫就有」的東西，多數只是「部分」**。

## 四、三個跨 repo 的共同模式

**模式一：護欄「有實作、無執行點」。** `x402-agent-payments` 的 `verifier/x402_v2/rate_limiter.py` 有完整的固定視窗限流實作（以每個 key 的首次請求錨定）、有 5 個單元測試，但**全 repo 零呼叫端**——它沒有接上任何請求路徑。而 `verifier/README.md` 的測試表把它列為「provider-side rate limiting」。也就是說：看文件你會以為有速率限制，看程式碼你會發現它是一座孤島。這種「有規則、無執行點」在盤點時最容易漏，因為它「有寫」。（**2026-09-24 更新**：README 已改述為「standalone helper；provider 本身不節流」，檔頭也改為 fixed window 的正確描述；孤島現況不變、已明示。）

**模式二：宣稱大於實作。** 兩個 repo 共 7 處，最明顯的一處是 `x402-agent-payments` 的 fail-closed：
- README 寫「Unknown networks, mismatched requirements, **missing token metadata** and v1 payloads all raise instead of degrading」。
- 程式碼是 `extra = requirement.extra or {}`，然後 `extra.get("name") or self.token_name`——缺 name/version 時**靜默套用預設值 `USD Coin`／`2`**，繼續驗簽，不 raise。

以碼為準：這是 degradation，不是 fail-closed。當應用層自己組 requirement 且沒帶 `extra` 時，驗證會落在與伺服器設定無關的預設 EIP-712 domain 上。**文件寫得比程式碼嚴格，會讓上線前的風險估算偏低**——這是我認為盤點最有價值的一類發現。

其餘落差包括：`docs-rag` 的錯誤訊息與註解自稱 multi-tenant（實際隔離是「一進程一語料」，查詢期沒有租戶身分、端點也沒有認證）；`requirements.txt` 註解宣稱檢索有「numpy → FTS-only」的第三層退路（實際上 numpy 匯入沒有 try/except，sqlite-vec 與 numpy 同時缺席時是拋錯）；`ragconfig.py` 宣告了 `LOGS_DIR` 但全 repo 零使用，計量實際寫進 checkout 的 `logs/`。

（**2026-09-24 更新**：上述 7 處已全數修補——x402 的 3 處見 `39aeffc`（metadata 改為 `invalid_requirement` fail-closed、`RateLimiter` 文件對齊、註解矛盾修正），docs-rag 的 3 處見 `b3168f4`（multi-tenant 用語、FTS-only 退路實作、計量與 trace 同落 `RAG_TRACE_DIR`／`LOGS_DIR`），另 1 處（replay 敘述易誤讀）已在兩份 README 明示「非 replay 防護」。逐項狀態與可重跑證據見 repo 的 `llm_security_audit.md` §十。）

**模式三：不可逆動作缺核准路徑。** B 組在兩個 repo 都只有「部分」。`x402-agent-payments` 的簽章是「簽下去就是可逆不了」的付款承諾，唯一的閘門是**呼叫端自報的 `max_amount_wei`**——沒有累計上限、沒有核准流程。`docs-rag` 的破壞性操作散落在 `client/sync_corpus.sh`（`rsync -a --delete`）、eval harness 的 `--fresh`（`rmtree`）、`refresh.sh`（`git reset --hard`），只有同步客戶端有 `--dry-run` 與目標 guard，其餘連預覽都沒有。

另外兩個值得記下的系統性空白：

- **資料面比控制面弱**：`docs-rag` 的控制面是 fail-closed（`RAG_CORPUS` 未設即 raise、設定壞 JSON 即 raise、同步目標不符即拒，都有測試），資料面卻是 fail-open——端點無認證、預設 `HOST=0.0.0.0`，而 MCP SDK 只在 host 是 loopback 時自動開啟 Host/Origin 檢查。**同一個系統兩種姿態**。
- **可稽核性不足**：兩個 repo 在 E1（可回放 trace）／E2（告警與負責人）／C6（保存期限）幾乎全數未實作。`x402-agent-payments` 完全沒有 logging；`docs-rag` 服務路徑每次呼叫只留一行計量（沒有 prompt、沒有輸出），出了事無法回放單次查詢。這是清單裡最系統性的一塊空白。

## 五、我不能假裝已覆蓋的部分

清單 G 組有一條「未實作項與其風險已明文記錄，不假裝已覆蓋」。以下是我這個盤點自己的邊界：

- **這不是一次資安審計**：沒有第三方參與、沒有滲透測試。我做的是一次**以清單為骨架的自查**：靜態讀碼為主，配合既有測試的實跑。
- **判定是 snapshot，不是 benchmark**：換一個部署環境（反向代理、容器、網路政策）會改變好幾個「未實作」項的實際風險。例如 `docs-rag` 的實際部署設定不在 repo 內，B 組的出網控制與 E4 的失敗姿態取決於它。
- **28 項不是全部**：它涵蓋我認為與「agent 有工具、有預算、有不可逆動作」最相關的面；供應鏈、模型本身的對齊、法遵不在其中。
- **我沒有修任何一個被盤點的 repo。**（**發布時**）落差先記錄、先講清楚，修不修是另一個決定——把「發現」和「修補」分開，是我在文件治理上的一貫做法。（**2026-09-24 更新**：本條已不成立——兩個 repo 的 7 處落差已修補並推送，明細見第四節註與 repo 的 `llm_security_audit.md` §十；原則是「先分開、後決定」，不是「永不修」。）

如果要在面試裡用這份清單，最誠實的說法是：**「我知道有 20 項完全沒做、23 項只做了一半，也知道每一項對應哪一條風險。」** 成熟度訊號來自能說出哪幾項還沒做、為什麼、風險多大，而不是宣稱全部已覆蓋。

## 六、你可以自己複核

| 主張 | 指令 |
|---|---|
| `x402-agent-payments` 測試（33＋15＋8） | `cd verifier && pytest -q`；`cd mcp && pytest -q`；`cd client && node --test` |
| `x402-agent-payments` 端到端 demo | `./scripts/demo.sh` → `402 → signed authorization → 200, in-process and chain-free` |
| `x402-agent-payments` 秘密掃描 | `./scripts/check-secrets.sh`；`./scripts/check-secrets.sh --history` |
| `docs-rag` 測試 | `pytest -q` → 33 passed |
| 「限流器零呼叫端」 | `grep -rn "RateLimiter" --include=*.py .` → 只命中自身與測試 |
| 「缺 token metadata 不 raise」 | 讀 `verifier/x402_v2/provider.py` 的 `extra.get("name") or self.token_name` |

清單本身（7 組 28 項）就在本文第一節，可以直接拿去用。
