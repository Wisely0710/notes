---
layout: default
title: "x402 × MCP：讓 agent 自己付錢"
---

# x402 × MCP：讓 agent 自己付錢

> 作者：Wisely（<https://github.com/Wisely0710>）｜2026-09
> 專案：<https://github.com/Wisely0710/x402-agent-payments>（MIT）——clone 之後跑 `./scripts/demo.sh`，30 秒跑完本文描述的全部流程（無鏈、無 facilitator、無真實金鑰）。

## 一個沒有帳號的買家

agent 已經會用工具了：讀檔、跑指令、呼叫 API。但只要 API 回一個 `402 Payment Required`，它就停在原地——不是因為不夠聰明，而是它沒有帳號、沒有信用卡，也沒有「這個月預算多少」的會計。

x402 把這個空格補起來：它是把 HTTP `402 Payment Required` 啟用成付款層的開放標準（Apache-2.0；2026 年 7 月起由 Linux Foundation 旗下的 x402 Foundation 治理），計價單位就是「單次 HTTP 請求」。搭配 MCP，agent 只需要多一個工具，就能自己完成一次付費取用。

這篇文章講我怎麼把兩邊接起來，以及五個我認為最值得留下的設計決策。所有程式碼在 [`x402-agent-payments`](https://github.com/Wisely0710/x402-agent-payments)，每個數字都可以自己複核。

## 協議：402 報價 → 簽章授權 → 驗證 → 交付

先花一分鐘把流程講清楚（規格：<https://docs.x402.org>）。

1. **賣方報價**。client 請求受保護資源，賣方回 `402`，付款要求放在 `PAYMENT-REQUIRED` 標頭（base64 的 `PaymentRequired`）。裡面的 `accepts[]` 是一份菜單：每筆是一個 `PaymentRequirements`——`scheme`、`network`（CAIP-2，例如 `eip155:8453`）、`amount`、`asset`、`payTo`、`extra`。同一個資源可以同時收多鏈、多資產。
2. **買方簽章**。client 從菜單挑一道自己能支援的，簽一張 **EIP-712 授權**——EVM 的 `exact` scheme 預設是 EIP-3009 的 `TransferWithAuthorization`（`from`／`to`／`value`／`validAfter`／`validBefore`／`nonce`），base64 後放進 `PAYMENT-SIGNATURE`，**重送同一個請求**。
3. **賣方驗證**。賣方自己驗，或委外給 facilitator（`POST /verify` 是唯讀的，不寫鏈上狀態）；驗證通過才執行 handler、回 `200`。
4. **結算是另一個步驟**。資金在 `/settle` 才真的移動，結果走 `PAYMENT-RESPONSE`。**簽章本身不是轉帳**，它是錢包簽下的付款承諾。

v2 與 v1 最大的差別在這裡：付款資訊全部走**標頭**，402 的回應主體留給服務自己用；`PAYMENT-REQUIRED`／`PAYMENT-SIGNATURE`／`PAYMENT-RESPONSE` 是 v2 的詞彙，v1 的 `X-PAYMENT` 已退役。

## 三個元件，一條可當場跑的線

我把它拆成三個元件，各自對應流程裡的一個角色：

| 元件 | 語言 | 角色 | 測試 |
|---|---|---|---|
| `verifier/` | Python | 賣方：產生 402 challenge、驗 `PAYMENT-SIGNATURE` | 33 |
| `client/` | TypeScript | 瀏覽器端：402 → 用注入錢包簽章 → 重試 | 15 |
| `mcp/` | Python | agent 端：MCP 工具 `fetch_paid_resource`，代 agent 付款 | 8 |

合計 56 個測試；client 的執行期依賴是 0。整個付費流程在 localhost 上跑完，**沒有鏈、沒有 facilitator、沒有真實金鑰**——而這條 demo 本身就是 CI 的一個 job。實際輸出長這樣（節錄）：

```text
[ agent] GET http://127.0.0.1:…/paid-resource (budget 1000 atomic units)
[seller] request without PAYMENT-SIGNATURE → 402
[seller]   accepts[0]: scheme=exact network=eip155:31337 amount=100 asset=0x3333…
[ agent]   signing EIP-712 TransferWithAuthorization value=100 to=0x4444… chainId=31337
[seller] retry with PAYMENT-SIGNATURE → verifying (x402_v2.provider)
[seller]   verified payer=0x1563… amount=100 network=eip155:31337
[seller]   settlement is out of scope here — no transaction is sent
[ agent] received: {"status": "paid", "data": "premium data"}
[  demo] 402 → signed authorization → 200, in-process and chain-free
```

## 五個設計決策

### 1. 只驗證、不碰錢

verifier 的輸出是一個純資料 DTO `VerifiedPayment`（payer、recipient、asset、atomic amount、network、nonce、有效時窗），**不含任何能動用資金的憑證**：不持有私鑰、不代送交易、不呼叫 facilitator，結算留給資源方。爆炸半徑最小化有個直接副作用：全部測試都能離線跑。

### 2. 官方 SDK 優先；自己寫的只有政策

x402 v2 的 wire 細節很多——標頭名稱、base64 codec、EIP-712 typed data、nonce、版本欄位。任何一處手寫都會長期漂移，所以這些全部取自官方 `x402` 套件；應用層只留三個決策點：**挑哪個 `accepts[]` 條目、預算規則、金鑰從哪來**。驗證端還多包一層 SDK-free 的 DTO，讓應用程式碼不必碰 SDK 型別。

一個具體的坑：以 Base 主網 USDC 為例，EIP-712 domain 必須是 `name="USD Coin"`、`version="2"`，與鏈上一致——這是最常見的失敗點，也正是我最不想自己維護的部分。

### 3. fail-closed，而且錯誤碼要能程式化

付款被拒的理由不只一種。六個穩定錯誤碼，每個都帶「下一步」：

| 錯誤碼 | 什麼時候出現 | 正確的下一步 |
|---|---|---|
| `invalid_version` | payload 不是 x402 v2 | 重新挑戰（拿新的 402） |
| `invalid_signature` | 簽章缺、格式錯或驗不過 | 請 client 重簽，不要重送同一份 |
| `invalid_time_window` | 過期、尚未生效或超過上限 | 換一個新的時間窗 |
| `authorization_mismatch` | 簽的 `to`／`value` 與要求不符 | 拒絕；client 簽了沒被要求的條件 |
| `requirement_mismatch` | client 挑錯了 `accepts[]` 條目 | 重新挑戰 |
| `network_mismatch` | 伺服器設定衝突（建構 provider 時就 raise） | 修設定 |

未知網路、缺 token metadata、不符的 requirement 一律 raise，不降級處理。**「拒絕」是這套系統的常態路徑之一**，所以它必須跟成功一樣好除錯。

### 4. 不做簽章快取：每次請求都真實驗簽

「這個簽章驗過了，快取起來」聽起來省 CPU，但快取鍵一旦沒把 `from` 綁進去，replay 就能撞到快取的成功結果。這裡選擇讓成功語意**不需要被信任**：不設簽章／nonce 快取，每次都以真實 ECDSA 重驗，並用測試釘住這個行為。成本是每請求多一次驗簽（本機 33 項測試 2.67 秒跑完），很划算。

### 5. agent 的唯一政策點：預算在簽章之前擋

讓 agent 自動付款，最可怕的不是簽章錯，而是「多打一個零」。MCP 工具因此要求 agent 自己宣告上限：`fetch_paid_resource(url, max_amount_wei)`；報價超過上限就拒絕——**連簽章都不做、也不送第二次請求**。簽章金鑰 lazy load，import server 模組不需要私鑰。

安全檢查的位置就是它的價值：放在不可逆動作（簽章）**之前**，而不是事後補救。

## 真實的失敗（比順利的部分值得留）

- **client 一開始沒有測試**。付款路徑靠人工驗證。後來補上 465 行 node:test：以 `node:http` 起 mock seller、用 stub 的 `window.ethereum` 當錢包，跑真 `fetch` 與真的 header codec——而且測的是 build 出來的 `dist/`，也就是套件的對外入口，不是內部實作。
- **secret 掃描一開始只看工作樹**。已刪掉但留在 git 歷史裡的值等於沒被檢查。改成三種模式（工作樹／staged／`--history`），並在腳本裡明寫「reflog 與 force-push 後的殘留掃不到」——**閘門的邊界要自己先講出來**，比「我的檢查都過」可信。
- **依賴解析也是可重現性的一部分**。Intel macOS 上某版 `cryptography` 沒有預編譯 wheel，`pip install` 直接失敗；在安裝指令裡 pin `cryptography<46` 繞過（CI 跑 ubuntu 不受影響）。這是環境限制、不是程式缺陷，但對「外部人 5 分鐘可跑」就是扣分項。

## 邊界（先講清楚，再談數字）

- 只實作 `exact` scheme 與 EVM（`eip155:*`）；`upto`、`batch-settlement`、鏈上結算、facilitator 呼叫都刻意列為非目標。
- mock seller 是測試替身，不是生產賣方；demo 的價值在於把協議流程變成 30 秒可驗證的操作。
- MCP 連線驗證用的是**官方 MCP Python SDK 的 client**（經 stdio 完成工具發現、付費呼叫與超預算拒絕）；我沒有以 Claude Code／Codex 本體實測。
- 未經第三方安全審計；限流是單機記憶體實作，多實例部署需要外部儲存。

## 自己複核

```bash
git clone https://github.com/Wisely0710/x402-agent-payments.git
cd x402-agent-payments
./scripts/demo.sh        # 約 30 秒：402 → 簽章 → 200（無鏈、無 facilitator、無真實金鑰）
```

- verifier 33 項（含錯誤碼與時間窗邊界值）、client 15 項（對 `dist/` 測、不需網路）、mcp 8 項（含官方 MCP client 的 stdio 端到端）。
- CI 五個 job（矩陣展開為七個實例）全 success——secret scan（含 git 歷史）、client、demo、verifier 3.11／3.12、mcp 3.11／3.12；最新一次 run [`35203413606`](https://github.com/Wisely0710/x402-agent-payments/actions/runs/35203413606)。
- README 裡的「真實輸出」就是 demo 的實際輸出，不是手寫的示意。

**一句話**：x402 規定「怎麼付」，MCP 提供「怎麼被呼叫」；接起來之後，agent 的付費能力其實是一個邊界問題——**誰在什麼時候被允許簽名**。我的答案是把簽章放在預算閘門之後、把結算留給錢包的主人，然後讓每一種拒絕都有名字。
