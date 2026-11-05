# IPsec VPN 實戰配置與避坑指南

本指南旨在解決企業在配置 IPsec VPN 時常遇到的效能瓶頸、斷線與不穩定問題。在 AI 普及的現在，許多自動生成的配置往往追求「理論上的最高安全強度」，卻忽略了硬體晶片的卸載能力（Hardware Offload）與協商過程中的運算成本。本專案以實際企業場景（如 FortiGate 101F 對接 401F，或 ADVPN 動態隧道）出發，提供兼顧安全性與極限效能的最佳實踐。

## 目錄 (Table of Contents)

1. [核心觀念：硬體晶片能力決定配置上限](#1-核心觀念硬體晶片能力決定配置上限)
2. [IKE 協定選擇：IKEv1 vs IKEv2](#2-ike-協定選擇ikev1-vs-ikev2)
3. [Diffie-Hellman (DH) Groups 評估：拋棄 MODP，全面擁抱 ECC](#3-diffie-hellman-dh-groups-評估拋棄-modp全面擁抱-ecc)
4. [Phase 1 與 Phase 2 加密現代化 (AES-GCM)](#4-phase-1-與-phase-2-加密現代化-aes-gcm)
5. [Rekey (金鑰更換) 的四大實戰陷阱與黃金原則](#5-rekey-金鑰更換的四大實戰陷阱與黃金原則)
6. [企業實戰配置速查矩陣 (Quick Reference)](#6-企業實戰配置速查矩陣-quick-reference)
7. [驗證與除錯指令 (以 FortiOS 為例)](#7-驗證與除錯指令-以-fortios-為例)

---

## 1. 核心觀念：硬體晶片能力決定配置上限

企業設定 VPN 的第一步並非對照資安合規表盲目開啟最高加密，而是確認設備底層 NPU / CP 晶片的硬體卸載（Hardware Offload）能力。

*   **木桶效應**：跨型號對接（例如邊緣端的 101F 與核心端的 401F）或跨廠牌對接時，必須以「硬體運算能力較弱」的一端為基準進行設定。
*   **硬體卸載失效的代價**：當選用設備晶片不支援的演算法（如某些型號不支援特定的橢圓曲線硬體加速），加密運算會退回 CPU 處理。在多隧道或大流量場景下，會導致 CPU 滿載、封包延遲、甚至系統崩潰。

## 2. IKE 協定選擇：IKEv1 vs IKEv2

無特殊跨廠牌相容性包袱時，**強制使用 IKEv2**。

*   **連線效率**：IKEv2 協商步驟更少，建立隧道速度更快。
*   **NAT 穿透 (NAT-T)**：IKEv2 原生內建支援，無需像 IKEv1 依賴額外的 RFC 實作。
*   **碰撞避免 (Collision)**：IKEv2 原生解決了兩端同時發起 Rekey 時產生的 SA 碰撞與懸空問題。

## 3. Diffie-Hellman (DH) Groups 評估：拋棄 MODP，全面擁抱 ECC

在動態建立 Shortcut（如 ADVPN）或頻繁 Rekey 的場景中，DH 運算成本是引發延遲的元兇。

| DH Group | 加密類型與強度 | 等效強度 | 軟體運算相對速度 (粗估) | 實戰評價 |
| :--- | :--- | :--- | :--- | :--- |
| **14** | MODP 2048 | ~112 bit | 很慢 (比 19 慢一個量級) | **應逐漸淘汰**。算力成本高，且強度落後現代標準。 |
| **15** | MODP 3072 | ~128 bit | 更慢 | **不建議使用**。 |
| **19** | ECP P-256 | ~128 bit | 基準 1x (最快的實用選項) | **企業標配首選**。完美平衡安全性、運算速度與硬體晶片相容性。 |
| **20** | ECP P-384 | ~192 bit | 約 19 的 2.5 ~ 3x 慢 | **高安控場景備用**。留有安全餘裕，效能仍優於任何 MODP。 |
| **21** | ECP P-521 | ~256 bit | 約 19 的 5 ~ 7x 慢 | **ADVPN 禁用**。運算太重，動態多隧道協商會瞬間榨乾 CPU。 |
| **31** | x25519 | ~128 bit | 比 19 快，實作不易出錯 | **需確認硬體支援**。演算法本身極快，但若設備無法硬體卸載會成為瓶頸。 |

## 4. Phase 1 與 Phase 2 加密現代化 (AES-GCM)

*   **Phase 1 (IKE SA)**：負責保護設備間的控制通道，資料量小。建議維持相容性高的 **AES-256-CBC + SHA-256**。
*   **Phase 2 (IPsec SA)**：負責保護實際傳輸的大量資料。
    *   **強烈建議改用 AES-256-GCM**。
    *   **關鍵避坑**：GCM 本身具備 AEAD（認證加密）特性，**在 Phase 2 不需要（也不能）額外設定 Authentication 演算法（如 SHA256）**。此配置能最大化發揮現代晶片（如 Fortinet NP7 / SoC4）的極限吞吐量。

## 5. Rekey (金鑰更換) 的四大實戰陷阱與黃金原則

盲目遵循「每小時換 Key」的合規要求，在企業大流量環境下容易引發網路抖動。

1.  **P1 與 P2 Lifetime 階層原則**：P1 的存活時間必須大於 P2（建議 2~3 倍以上）。嚴禁兩者設為相同時間，避免同時過期引發控制封包競爭。
2.  **大流量殺手「Byte Limit」**：多數設備預設開啟「傳輸量觸發 Rekey」。在 1Gbps 等級的 S2S 備份場景下，幾秒鐘就會觸發一次換 Key。**強烈建議在 P2 關閉 Byte Limit（設為 0），僅依賴 Time-based 觸發。**
3.  **PFS (Perfect Forward Secrecy) 的連鎖成本**：P2 開啟 PFS 可確保前向機密，但每次換 Key 都要重新進行 DH 計算。若 P2 Lifetime 短且搭配笨重的 DH Group 14，將產生週期性的 CPU 尖峰。解法：全面轉向 DH Group 19。
4.  **錯開計時器避免碰撞 (針對 IKEv1)**：若受限環境只能用 IKEv1，請在兩端設定不對稱的 Lifetime（如總部 28800s，分支 27000s），由單一端固定發起協商，避免 SA Glare。

## 6. 企業實戰配置速查矩陣 (Quick Reference)

| 設定參數 | 傳統/AI 常見預設 (應棄用) | 企業實戰最佳解 (推薦) | 核心效益 |
| :--- | :--- | :--- | :--- |
| **IKE Version** | IKEv1 | **IKEv2** | 避免碰撞，協商效率高 |
| **Phase 1 DH** | Group 14 (MODP 2048) | **Group 19 (ECP 256)** | 算力大降，等效強度達標 |
| **Phase 1 Lifetime** | 28800s 或與 P2 相同 | **86400s (24h)** | 穩定控制層，減少重建 |
| **Phase 2 Cipher** | AES-256-CBC + SHA256 | **AES-256-GCM (不設 Auth)** | 完美啟用硬體卸載，極致效能 |
| **Phase 2 PFS** | 開啟 (Group 14) | **開啟 (與 P1 一致，Group 19)** | Rekey 無感切換 |
| **Phase 2 Lifetime** | 3600s + Byte Limit (512MB) | **28800s (8h)，Byte Limit=0** | 避免大流量引發頻繁 Rekey 癱瘓設備 |

## 7. 驗證與除錯指令 (以 FortiOS 為例)

配置完成後，必須透過 CLI 確認流量是否真正交由硬體加速，而非停留在 CPU 運算。

*   **檢查硬體卸載標記 (npu_flag)**:
    ```bash
    diagnose vpn tunnel list
    ```
    *(尋找輸出中的 npu_flag 確認硬體卸載狀態)*

*   **確認 ESP 封包處理狀態**:
    ```bash
    diagnose vpn ipsec status
    ```
    *(驗證加解密是否由 NP/CP 處理)*
