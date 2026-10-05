# macOS Caps Lock Delay: Mechanics Analysis & Decoupling Workaround
> macOS Caps Lock 延遲痛點：底層機制解析與單雙擊解耦方案

**📜 License:** MIT | **💻 Platform:** macOS

---

### 專案結論 (TL;DR)
本專案透過 Karabiner-Elements 替換 macOS 原生「時間長度判定」機制，改以「敲擊次數判定」解耦輸入法切換與大寫鎖定功能，規避系統底層的防誤觸延遲。同時探討不同實體鍵盤軸體在極限參數下的設定差異與系統權限邊界。

👉 **如果你不需要看原理，只想馬上解決延遲問題，請直接點擊跳轉至：[3. 部署與設定指南 (Installation & Setup)](#3-部署與設定指南-installation--setup) 複製代碼。**

---

## 1. 根本痛點 (Why)
macOS 原生將 Caps Lock 鍵強行綁定了「按壓時間長短」來判定三種狀態：

* **極短暫按壓 (約 < 80ms)**：系統當成誤觸（直接丟棄訊號，按了沒反應）。
* **短按 (約 80ms - 250ms)**：觸發中英輸入法切換。
* **長按 (約 > 250ms)**：觸發英文大寫鎖定（綠燈亮起）。

這種底層設計，導致打字速度快的文字工作者高頻切換輸入法時，手指敲擊節奏快於系統判定閾值，產生不可避免的卡頓與切換失敗。

---

## 2. 解決方案演進與邊界探測 (Evolution & Trade-offs)
> ⚠️ **重要說明**：本節僅作為不同方案的技術規格與極限比較。本專案**不贅述**這些過渡方案的具體實作過程。如果你對特定方案感興趣，請自行依據表格中的「關鍵字」上網求解。

| 方案級別 | 執行邏輯 (關鍵字) | 改善程度 | 系統權限限制與妥協 (Trade-offs) |
| :--- | :--- | :--- | :--- |
| **方案 D (原生偏方)** | `hidutil` 歸零延遲 / 緩慢鍵 `Slow Keys` | 零改善 (甚至惡化) | 未脫離時間判定框架。會導致 Caps Lock 極高頻誤判，或引發全域鍵盤嚴重漏字。 |
| **方案 C (完全替換)** | 停用 Caps Lock，改用 `Control + Space` | 徹底消除延遲 | 必須強制重新建立肌肉記憶。 |
| **方案 B (功能降級)** | Karabiner 將 Caps Lock 改為純輸入法鍵 | 徹底消除延遲 | 喪失大小寫鎖定功能，按鍵利用率降低。 |
| **方案 A (時間解耦)** | Karabiner 拆分：雙擊切換、長按鎖定 | 顯著改善 | 仍受制於長按閾值，具等待遲滯感。 |
| **進階方案 A (本專案實作)** | **雙擊切換輸入法、單擊強制輸出 Caps Lock** | **絕對零延遲** | 中文模式下單擊僅能強制切換為小寫英文，需配合 `Shift` 輸出大寫。 |
| **終極方案 (硬體韌體)** | 客製化鍵盤寫入 `VIA / QMK` 巨集 | 完美繞過限制 | 具硬體門檻，筆電內建鍵盤無法適用。 |

---

## 3. 部署與設定指南 (Installation & Setup)

### 步驟 1：前置作業（檢查權限）
1. 請確保電腦已下載並安裝 [Karabiner-Elements](https://pqrs.org)。
2. 打開 macOS 的 `系統設定` > `隱私權與安全性`。
3. 確保在 **「輸入監聽 (Input Monitoring)」** 與 **「輔助使用 (Accessibility)」** 中，都已經勾選允許 Karabiner 開啟。

### 步驟 2：打開設定檔
1. 打開你的 Mac 終端機 (Terminal) 或你慣用的文字編輯器（如 VS Code / Cursor）。
2. 開啟 Karabiner 的預設設定檔案，路徑在：`~/.config/karabiner/karabiner.json`

### 步驟 3：貼入代碼（請注意 JSON 語法防呆）
1. 在該檔案中，利用關鍵字搜尋（`Cmd + F`）找到 `"rules": [` 這個陣列。
2. 將下方的 JSON 代碼**完整複製**，貼進 `"rules": [` 括號裡面的最前排。
3. **⚠️ 語法防呆注意**：如果你的 `rules` 陣列原本就有其他規則，請記得在貼入的代碼最尾端的 `}` 後方**加上一個英文逗號 `,`**，否則格式出錯會導致 Karabiner 無法讀取。

```json
{
  "description": "F19 (Caps Lock): 雙擊切換輸入法, 單擊輸出大/小寫 (60ms 矮軸特化版)",
  "manipulators": [
    {
      "type": "basic",
      "conditions": [
        {
          "type": "variable_if",
          "name": "caps_double_tap",
          "value": 1
        }
      ],
      "from": {
        "key_code": "f19",
        "modifiers": { "optional": ["any"] }
      },
      "to": [
        { "key_code": "spacebar", "modifiers": ["left_control"] },
        { "set_variable": { "name": "caps_double_tap", "value": 0 } }
      ]
    },
    {
      "type": "basic",
      "from": {
        "key_code": "f19",
        "modifiers": { "optional": ["any"] }
      },
      "to": [
        { "set_variable": { "name": "caps_double_tap", "value": 1 } }
      ],
      "to_delayed_action": {
        "to_if_invoked": [
          { "key_code": "caps_lock" },
          { "set_variable": { "name": "caps_double_tap", "value": 0 } }
        ],
        "to_if_canceled": [
          { "set_variable": { "name": "caps_double_tap", "value": 0 } }
        ]
      },
      "parameters": {
        "basic.to_delayed_action_delay_milliseconds": 60
      }
    }
  ]
}
```
4. 儲存檔案（`Cmd + S`）。Karabiner 會在背景自動偵測並立刻生效，不需要重啟軟體。

---

## 4. 最終操作邏輯與參數調校 (User Experience & Tuning)

導入設定後，Caps Lock 的時間判定機制已被完全抹除，操作邏輯會完全轉變為**次數判定**：

| 實體敲擊動作 | 當前輸入法狀態 | 系統執行結果 |
| --- | --- | --- |
| **連按兩下 (雙擊)** | 任何狀態 | 完全攔截原本的訊號，發送 `Control + Space` 切換中/英輸入法。 |
| **按一下 (單擊)** | 英文語法狀態 | 正常切換大小寫鎖定 (Caps Lock On/Off)。 |
| **按一下 (單擊)** | 中文語法狀態 | 受限 macOS 底層權限機制，會強制輸出英文小寫。想輸入大寫需配合 `Shift` 鍵。 |

### 硬體軸體特性與判定閾值分析 (Threshold Tuning)

代碼中的 `"basic.to_delayed_action_delay_milliseconds": 60` 代表「系統判斷你是不是連按兩下」的等待時間。這個參數非常取決於你的**鍵盤物理回彈速度**，你可以依據下圖與表格進行微調：

<img width="600" alt="axis_delay_comparison" src="https://github.com/user-attachments/assets/7461eb9e-3d66-4858-9759-e792c6d229bb" />

| 鍵盤軸體類型 | 物理行程特性 | 建議判定閾值 | 調整邏輯說明（好懂版） |
| --- | --- | --- | --- |
| **矮軸 / 線性軸**<br>(如 Keychron K3 Max) | 鍵帽按下去的物理行程極短 (~2.5mm)，按起來完全沒有阻力，回彈速度極快。 | **50ms - 70ms** | 手指從按下去到完全放開可以在 50 毫秒內搞定。把數值壓低到 60ms 可以讓你的中英切換達到神經反射級的「絕對零延遲」。 |
| **標準高度機械軸**<br>(如青軸/茶軸/常規紅軸) | 鍵帽比較高，總行程長 (~4.0mm)，按下去比較深。 | **80ms - 100ms** | 因為鍵帽回彈需要走比較長的路徑。如果這個數值調得太低（例如設 60ms），系統會以為你只是按得比較慢的「單擊」，導致雙擊切換輸入法失效。 |

---

## License

MIT License
