# AGENTS.md

## 專案目標

這個專案要做一套以 Bluetooth beacon 定位為基礎的數位化定向越野遊戲裝置。
最終情境是玩家使用 Android 手機在場域中移動，手機與 Raspberry Pi 5 裝置互相發送與接收固定格式的 BLE beacon。當玩家手機靠近指定裝置，Raspberry Pi 偵測到足夠強的手機 beacon 後，啟動相機互動流程，讓玩家完成拍照或手勢任務。

Raspberry Pi 裝置除了作為遊戲檢查點，也會結合相機、影像辨識與馬達控制，讓鏡頭能自動追蹤玩家並居中。玩家完成指定手勢後，裝置會拍照並儲存任務結果。

## 目前完成狀態

- `camera.py` 已完成 USB 相機互動流程。
- 已支援人臉偵測與自動水平居中。
- 已支援多人居中：偵測到多張臉時，會使用所有人臉中心點的幾何中心作為旋轉依據；單人時維持原本單人中心邏輯。
- 已支援 YA 手勢偵測與倒數拍照。
- 如果鏡頭還在旋轉居中，系統不會拍照，會等居中完成後才允許 YA 倒數。
- 拍照結果會存到 `photos/`，檔名格式為 `ya_YYYYMMDD_HHMMSS.jpg`。
- `rpi_beacon_camera.py` 已整合 Raspberry Pi 端 BLE 與相機流程。
- Raspberry Pi 會發送固定 Pi beacon，讓手機端之後可以接收。
- Raspberry Pi 會掃描手機發出的固定格式 beacon。
- 手機 beacon RSSI 達到門檻且連續命中指定次數後，會啟動相機 session。
- `--preview` 可開啟相機預覽視窗。
- 預覽模式下，因為 OpenCV Qt 視窗與 BlueZ/GLib event loop 在同一流程內容易衝突，會暫停 Pi beacon，等相機 session 結束後再重新開始廣播。
- 非預覽模式下，整合流程可維持 headless 運作。
- `bluez_venv.py` 已處理 virtualenv 內找不到系統 `dbus`/`gi` 套件的問題。
- `le_advertiser.py`、`le_scanner.py` 是已跑通過的 BLE 範例，可作為調整 beacon 格式或 debug 的參考。

## 重要檔案

- `rpi_beacon_camera.py`
  - Raspberry Pi 主程式。
  - 負責 Pi beacon 廣播、手機 beacon 掃描、RSSI 觸發與相機 session 啟動。
- `camera.py`
  - 相機、人臉偵測、多人居中、馬達控制、YA 手勢偵測與拍照流程。
- `bluez_venv.py`
  - 讓 Python virtualenv 可以使用 Raspberry Pi OS 透過 apt 安裝的 `dbus`/`gi`。
- `le_advertiser.py`
  - BLE 廣播範例。
- `le_scanner.py`
  - BLE 掃描範例，目前未納入 git 追蹤。
- `haarcascade_frontalface_default.xml`
  - OpenCV Haar cascade 人臉偵測模型。
- `requirements.txt`
  - Python pip 套件需求。

## 目前 beacon 設定

目前 Raspberry Pi 與手機端預期使用相同 manufacturer data 作為測試格式。

- Pi local name: `RPi_112360104`
- Manufacturer ID: `0xFFFF`
- Pi beacon data: `0x00110044`
- Phone beacon data: `0x00110044`
- RSSI trigger default: `-60 dBm`
- Required strong hits default: `3`
- Camera cooldown default: `3` 秒
- Camera session timeout default: `15` 秒

之後 Android 端實作時，需要讓手機送出相同格式的 manufacturer data，或同步修改 Raspberry Pi 端的 `PHONE_BEACON_DATA`。

## 執行方式

在 Raspberry Pi 5 上執行前，先安裝系統套件：

```bash
sudo apt install python3-dbus python3-gi
```

Python 套件安裝：

```bash
pip install -r requirements.txt
```

執行整合流程，無預覽：

```bash
python rpi_beacon_camera.py
```

執行整合流程，開啟預覽：

```bash
python rpi_beacon_camera.py --preview
```

常用參數：

```bash
python rpi_beacon_camera.py --rssi -55 --strong-hits 3 --cooldown 3 --camera-timeout 15
```

只測試相機互動流程：

```bash
python camera.py
```

## 後續工作

- 實作 Android 手機端 BLE beacon 發射。
- 實作 Android 手機端接收 Raspberry Pi beacon，作為玩家定位或關卡提示依據。
- 決定正式 beacon payload 格式，避免目前 Pi/Phone 使用相同 data 時在多裝置場域中難以區分角色。
- 建立每個檢查點的 ID、關卡狀態與任務規則。
- 設計手機端 UI，顯示目前關卡、附近檢查點、任務提示與完成狀態。
- 增加任務完成後的資料記錄，例如照片路徑、完成時間、玩家 ID、檢查點 ID。
- 評估是否要把 BLE 廣播/掃描與相機預覽拆成不同 process，降低 OpenCV Qt 與 BlueZ/GLib event loop 衝突。
- 測試多人場景下的人臉誤偵測、遮擋與鏡頭居中的穩定度。

## 開發注意事項

- Raspberry Pi 端目前以 BlueZ、bluezero、bleak 實作 BLE 功能。
- virtualenv 內如果出現 `No module named dbus`，確認 `python3-dbus` 已透過 apt 安裝，並確認程式有先呼叫 `enable_system_dbus_path()`。
- 修改 BLE 流程時，優先保留 `rpi_beacon_camera.py` 的 headless 模式可用。
- 修改相機流程時，要維持「馬達正在居中時不能拍照」這個規則。
- 修改多人居中時，要保留單人情境與原本操作手感。
- 不要把測試照片、cache、virtualenv 或本機硬體輸出檔納入 commit。
