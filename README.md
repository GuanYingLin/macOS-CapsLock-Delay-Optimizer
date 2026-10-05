# macOS Caps Lock Delay: Mechanics Analysis & Decoupling Workaround
> macOS Caps Lock 延遲痛點：底層機制解析與單雙擊解耦方案

**📜 License:** MIT | **💻 Platform:** macOS

---



---

### 專案結論 (TL;DR)
本專案透過 Karabiner-Elements 替換 macOS 原生「時間長度判定」機制，改以「敲擊次數判定」解耦輸入法切換與大寫鎖定功能，規避系統底層的防誤觸延遲。同時探討不同實體鍵盤軸體在極限參數下的設定差異與系統權限邊界。

---

## 1. 根本痛點 (Why)
macOS 將原生 Caps Lock 鍵設計為單一按鍵，並依賴「按壓時間長短」判定三種狀態（註：蘋果並未公開精確參數，以下時間閾值為開發者社群反向測量之經驗估值）：

* **極短暫按壓 (約 < 80ms)**：防誤觸機制（系統直接丟棄訊號）。
* **短按 (約 80ms - 250ms)**：觸發中英輸入法切換。
* **長按 (約 > 250ms)**：觸發英文大寫鎖定。

此依賴時間維度的判定機制，導致文字工作者在高頻切換輸入法時，常因手指敲擊節奏快於系統判定閾值，產生不可避免的切換卡頓與指令誤判。

---

## 2. 解決方案演進與邊界探測 (Evolution & Trade-offs)
本專案經歷多輪測試，探索從系統偏方、純軟體修改到實體硬體層級的各項解法與妥協方案：

| 方案級別 | 執行邏輯 | 改善程度 | 系統權限限制與妥協 (Trade-offs) |
| :--- | :--- | :--- | :--- |
| **方案 D (原生偏方)** | 透過終端機 `hidutil` 歸零延遲，或開啟系統「緩慢鍵 (Slow Keys)」。 | 零改善 (甚至惡化) | 未脫離時間判定框架。會導致 Caps Lock 極高頻誤判，或引發全域鍵盤嚴重漏字。 |
| **方案 C (完全替換)** | 停用 Caps Lock 切換，改用原生 `Control + Space`。 | 徹底消除延遲 | 需重新建立肌肉記憶。 |
| **方案 B (功能降級)** | 透過 Karabiner 將 Caps Lock 變更為純輸入法切換鍵。 | 徹底消除延遲 | 喪失大小寫鎖定功能，按鍵利用率降低。 |
| **方案 A (時間解耦)** | Karabiner 拆分：雙擊切換、長按鎖定大小寫。 | 顯著改善 | 仍受制於長按閾值，具等待遲滯感。 |
| **進階方案 A (本專案實作)** | 雙擊切換輸入法、單擊強制輸出 Caps Lock 訊號。 | 絕對零延遲 | 中文模式下單擊僅能強制切換為小寫英文，需配合 `Shift` 輸出大寫。 |
| **終極方案 (硬體韌體)** | 採用支援 VIA/QMK 的外接鍵盤寫入巨集。 | 完美繞過限制 | 具硬體門檻，筆電內建鍵盤無法適用。 |

---

## 3. 部署與設定指南 (Installation & Setup)

### 前置作業
請確認系統已安裝 [Karabiner-Elements](https://pqrs.org)，並已授予必要的 macOS 隱私權限（輸入監聽與輔助使用）。

### 設定步驟
1. 開啟終端機或純文字編輯器，前往 Karabiner 設定檔路徑：`~/.config/karabiner/karabiner.json`
2. 找到 `profiles` > `complex_modifications` > `rules` 陣列。
3. 將以下 JSON 規則完整貼入 `rules` 陣列中並存檔。Karabiner 將自動重載設定。

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

---

## 4. 最終操作邏輯與參數調校 (User Experience & Tuning)

導入上述設定後，Caps Lock 的時間判定機制已被完全抹除，操作邏輯轉變如下：

| 實體敲擊動作 | 當前輸入法狀態 | 系統執行結果 |
| --- | --- | --- |
| **雙擊 (Double Tap)** | 任何狀態 | 攔截實體訊號，發送 `Control + Space` 切換中/英輸入法。 |
| **單擊 (Single Tap)** | 英文語法 | 正常切換大小寫鎖定 (Caps Lock On/Off)。 |
| **單擊 (Single Tap)** | 中文語法 | 受限 macOS 權限機制，強制輸出英文小寫。欲輸入大寫需配合 `Shift` 鍵。 |

### 硬體軸體特性與判定閾值分析 (Threshold Tuning)

上述 JSON 程式碼中的 `"basic.to_delayed_action_delay_milliseconds": 60` 為雙擊等待極限。此參數必須與鍵盤軸體物理特性適配：

<img width="600" alt="axis_delay_comparison" src="https://github.com/user-attachments/assets/7461eb9e-3d66-4858-9759-e792c6d229bb" />


| 鍵盤軸體類型 | 物理行程特性 | 建議判定閾值 | 判定邏輯說明 |
| --- | --- | --- | --- |
| **矮軸 / 線性軸** (如 Keychron K3 Max) | 總行程極短 (~2.5mm)，觸發淺，無段落阻力。 | **50ms - 70ms** | 按壓至完全回彈可於 50ms 內完成。壓低至 60ms 可大幅壓縮判定窗口，達成神經反射級切換。 |
| **標準高度機械軸** | 總行程長 (~4.0mm)，觸發點較深。 | **80ms - 100ms** | 鍵帽回彈耗時較長。若閾值設定過低，系統易將常規單擊誤判為黏滯，導致雙擊指令失效。 |

---

## License

MIT License
