[English](README.md) · **繁體中文**

# 反應時間對決 (Reaction Time Duel)
## 本專案獲選於 Hanze Open Day 展示！耶！
<img width="2048" height="1536" alt="IMG_2032" src="https://github.com/user-attachments/assets/0c61f458-73cb-4014-977f-d88b95b5550a" />

一款使用 ESP32 與 ESP8266 微控制器打造的多人競爭型反應時間遊戲，透過 ESP-NOW 進行無線通訊。最多支援 4 名玩家使用客製化無線搖桿控制器，在兩種遊戲模式中進行 5 回合的競賽。

## 遊戲模式

### 反應模式 (Reaction Mode)
NeoPixel 燈環會循環變換隨機顏色。當 LED 燈號定格為黃色時 — 按下你的按鈕！反應時間最快的玩家贏得該回合。太早按會受到懲罰。

### 搖晃模式 (Shake Mode)
在 3-2-1 倒數後，搖晃你的搖桿以達到目標次數（10、15 或 20 次搖晃）。中央燈環倒數顯示剩餘時間。最先完成者獲勝。

> 模式透過 **shuffle bag**（隨機袋抽籤機制）進行分配 — 確保兩種模式都會出現過後，任一模式才會再次重複。

## 硬體

| 元件 | MCU | 數量 | 角色 |
|-----------|-----|-----|------|
| 主控制器 (Host Controller) | ESP32 (DevKit-C) | 1 | 遊戲邏輯、NeoPixel 燈環、環境氛圍 LED 燈條、I2S 音訊 |
| 搖桿 (Joystick) | ESP8266 (ESP-12F) | 最多 4 | 按鈕輸入、加速度計、震動馬達 |
| 顯示器 (Display) | ESP32/ESP8266 | 1 | 遊戲狀態顯示（選用，接收 ESP-NOW） |

### 主控制器 (Host Controller)
- **5 個 NeoPixel 燈環**（每個 12 顆 LED，共 60 顆）— 玩家狀態與遊戲動畫
- **89 顆 LED 的 WS2812B 氛圍燈條** — 循環播放 6 種動畫（rainbow、sparkle、meteor、color chase、breathing、fire）
- **I2S MP3 音訊** — SPIFFS 上存有 24 個音效檔案，用於語音播報、倒數及音效
- **PWM 音量控制** — 透過擴大機 GAIN 腳位實現硬體增益控制

### 搖桿控制器 (Joystick Controller)
- **按鈕** (GPIO14, active LOW) — 反應輸入，採用 IRAM 中斷以達到微秒級精度
- **MPU-6050 加速度計** — 搖晃偵測，具備高通濾波遲滯判定（X+Z 軸）
- **震動馬達** — 倒數讀秒、GO 信號及完成時的觸覺回饋

## 系統架構

```
                         ┌──────────────────────┐
                         │     ESP32 Host        │
                         │    (Game Logic)       │
                         │                      │
                         │  NeoPixel Rings ×5   │
                         │  WS2812B Strip (89)  │
                         │  I2S Audio + Amp     │
                         │  SPIFFS (24 MP3s)    │
                         └──────────┬───────────┘
                                    │ ESP-NOW (ch 6)
               ┌─────────────────────┼─────────────────────┐
               │                     │                     │
         ┌─────┴──────┐       ┌─────┴──────┐        ┌─────┴──────┐
         │ Joystick 1 │       │ Joystick 2 │  ...   │ Joystick 4 │
         │  ESP8266   │       │  ESP8266   │        │  ESP8266   │
         │            │       │            │        │            │
         │  Button    │       │  Button    │        │  Button    │
         │  MPU-6050  │       │  MPU-6050  │        │  MPU-6050  │
         │  Vibration │       │  Vibration │        │  Vibration │
         └────────────┘       └────────────┘        └────────────┘
                                    │
                               ┌─────┴──────┐
                               │  Display   │
                               │ (optional) │
                               └────────────┘
```

## 遊戲流程

```
IDLE ──> JOIN ──> COUNTDOWN ──> REACTION / SHAKE ──> COLLECT ──> RESULTS ──┐
  ^                                                                        │
  │  (after 5 rounds)    FINAL_WINNER <────────────────────────────────────┘
  └──────────────────────────┘
```

1. **待機 (Idle)** — 彩虹動畫，播放「press to join」語音提示
2. **加入 (Join)** — 依序提示玩家加入（P1 → P2 → P3 → P4）。每個位置在其 NeoPixel 燈環上閃爍 5 秒。任何尚未綁定的搖桿皆可按下按鈕認領該位置。至少需要 2 名玩家。
3. **模式選擇 (Mode Selection)** — Shuffle bag 挑選反應模式 (Reaction) 或搖晃模式 (Shake)。每場遊戲中每種模式的首次說明語音僅播放一次。
4. **倒數 (Countdown)** — 搖晃模式具備 3-2-1 倒數，伴隨同步音效、NeoPixel 閃爍及觸覺震動。反應模式跳過倒數，在發出 GO 信號前採用隨機延遲（3 秒、5 秒或 7 秒）。
5. **遊戲進行 (Play)** — 反應模式：LED 定格為黃色 = 立即按下。搖晃模式：競速達到目標次數。
6. **收集結果 (Collect)** — 透過 ESP-NOW 接收結果。在淘汰逾時玩家前，會對慢速玩家顯示黃色警告階段。
7. **回合結果 (Results)** — 顯示時間 3 秒，接著顯示贏家與得分 3 秒。
8. **最終贏家 (Final Winner)** — 進行 5 回合後，宣布總冠軍並播放勝利號角音樂。

## 專案結構

```
ReactionTimeDuel/
├── README.md
├── .gitignore
│
├── ReactionTimerHost/              # ESP32 Host Controller
│   ├── platformio.ini
│   ├── enable_ccache.py            # Build speed optimization
│   ├── src/
│   │   └── main.cpp                # Game state machine, NeoPixel, strip animations
│   ├── include/
│   │   ├── Protocol.h              # Packet format, CRC8, device IDs, commands
│   │   ├── GameTypes.h             # Constants, timing, player struct, NeoPixel config
│   │   └── AudioManager.h          # Non-blocking MP3 queue, I2S output, PWM volume, sound defs
│   └── data/                       # 24 MP3 files uploaded to SPIFFS
│       ├── beep.mp3, click.mp3, error.mp3
│       ├── get_ready.mp3, react_inst.mp3, will_shake.mp3
│       ├── player1.mp3 ... player4.mp3
│       ├── one.mp3, two.mp3, three.mp3, ten.mp3, fifteen.mp3, twenty.mp3
│       ├── reaction.mp3, shake.mp3, fastest.mp3, wins.mp3
│       └── victory.mp3, gameover.mp3, press_join.mp3, ready.mp3
│
└── ReactionTimerSlave/             # ESP8266 Joystick Controllers (×4)
    ├── platformio.ini              # Multi-env: stick1, stick2, stick3, stick4
    ├── enable_ccache.py
    ├── src/
    │   └── main.cpp                # Joystick state machine, shake detection, timing
    └── include/
        ├── Protocol.h              # Shared protocol (core commands, no display cmds)
        └── GameTypes.h             # Joystick-only constants (timeouts)
```

## 建置與燒錄

### 前置需求
- [PlatformIO](https://platformio.org/) IDE 或 CLI
- ESP32 DevKit-C 與 ESP-12F 開發板
- 用於燒錄程式的 USB 傳輸線

### 主控制器 (ESP32)

```bash
cd ReactionTimerHost
pio run -t upload          # Flash firmware
pio run -t uploadfs        # Upload MP3 audio files to SPIFFS
```

### 搖桿 (ESP8266)

每支搖桿都有專屬的建置環境與各自的 `MY_ID` 旗標：

```bash
cd ReactionTimerSlave
pio run -e stick1 -t upload    # Joystick 1 (MY_ID=0x01)
pio run -e stick2 -t upload    # Joystick 2 (MY_ID=0x02)
pio run -e stick3 -t upload    # Joystick 3 (MY_ID=0x03)
pio run -e stick4 -t upload    # Joystick 4 (MY_ID=0x04)
```

## 腳位配置

### 主控端 (ESP32)

| 腳位 | 功能 |
|-----|----------|
| GPIO4 | NeoPixel 燈環資料（5 個燈環，60 顆 LED） |
| GPIO16 | WS2812B 氛圍燈條資料（89 顆 LED） |
| GPIO25 | I2S DOUT（音訊資料） |
| GPIO26 | I2S BCLK（位元時脈） |
| GPIO27 | I2S LRC（字元選擇 word select） |
| GPIO33 | 擴大機 GAIN（PWM 音量控制） |

### 搖桿端 (ESP8266)

| 腳位 | 功能 |
|-----|----------|
| GPIO14 | 按鈕輸入（active LOW，內部上拉） |
| GPIO12 | 震動馬達輸出 |
| GPIO4 | I2C SDA (MPU-6050) |
| GPIO5 | I2C SCL (MPU-6050, 400kHz) |

## 通訊協定

所有裝置皆透過 **頻道 6 上的 ESP-NOW** 以 7 位元組封包進行通訊：

| 位元組 | 欄位 | 說明 |
|------|-------|-------------|
| 0 | `START` | 固定為 `0x0A` |
| 1 | `DEST_ID` | 目的地（0x00=Host, 0x01-04=Sticks, 0x10=Display, 0xFF=Broadcast） |
| 2 | `SRC_ID` | 發送端 ID |
| 3 | `CMD` | 命令位元組 |
| 4 | `DATA_HIGH` | 資料高位元組 |
| 5 | `DATA_LOW` | 資料低位元組 |
| 6 | `CRC` | 針對位元組 0-5 計算的 CRC8（多項式 0x8C） |

### 主要命令

| 命令 | 十六進位 | 方向 | 說明 |
|---------|-----|-----------|-------------|
| `CMD_REQ_ID` | `0x0D` | Stick → Host | 請求加入（資料編碼包含韌體版本） |
| `CMD_OK` | `0x0B` | Host → Stick | 確認加入（資料 = slot） |
| `CMD_ACK` | `0x0E` | 雙向 | 確認前一條命令 |
| `CMD_GAME_START` | `0x21` | Host → Stick | 回合開始（高位元組=模式，低位元組=參數） |
| `CMD_GO` | `0x22` | Host → Stick | GO 信號 — 開始計時 |
| `CMD_VIBRATE` | `0x23` | Host → Stick | 震動（`0xFF`=GO，其餘為持續時間 x 10ms） |
| `CMD_IDLE` | `0x24` | Host → Stick | 返回待機狀態 |
| `CMD_COUNTDOWN` | `0x25` | Host → Stick | 倒數讀秒（3, 2, 1） |
| `CMD_REACTION_DONE` | `0x26` | Stick → Host | 反應時間（毫秒，`0xFFFF` = 懲罰） |
| `CMD_SHAKE_DONE` | `0x27` | Stick → Host | 搖晃時間（毫秒，`0xFFFF` = 逾時） |
| `CMD_SHAKE_PROGRESS` | `0x28` | Stick → Host | 搖晃進度里程碑（每 5 次搖晃） |

主控端亦會向選用的顯示器發送顯示命令（`0x30`-`0x3E`），以呈現即時遊戲狀態。

## 技術特點

- **微秒級反應計時** — 透過 `IRAM_ATTR` 下降邊緣中斷捕捉按鈕按下瞬間；時間計算為相對於 GO 信號的 `micros()` 差值
- **非阻塞式架構** — 採用 NeoPixelBus 搭配 ESP32 RMT DMA 實現無雜訊的 LED 輸出；音訊佇列具備可設定的音效間隔；遊戲迴圈中絕無 `delay()`
- **高通濾波搖晃偵測** — EMA 低通濾波器（alpha=1/64）消除重力；遲滯狀態機計算 X+Z 軸上的完整推回週期
- **即時搖晃進度** — 搖桿每搖晃 5 次便透過 `CMD_SHAKE_PROGRESS` 回報里程碑，使主控端的 NeoPixel 燈環能呈現即時進度條動畫
- **Shuffle Bag 模式選擇** — 反應模式與搖晃模式皆出現過後才會重複，避免連續出現相同模式
- **環境氛圍燈條** — 89 顆 LED 的 WS2812B 燈條透過第二個 RMT 通道循環播放 6 種程序化動畫（rainbow、sparkle、meteor rain、color chase、breathing、fire）
- **PWM 音量控制** — 擴大機 GAIN 腳位由 25kHz LEDC PWM 驅動，實現平滑的類比音量調節
- **韌體版本控管** — 協定在加入封包中包含韌體版本（V3.0.0）以進行相容性檢查
- **可靠傳輸** — 主控端針對關鍵命令使用 ACK 表，最多重試 3 次（間隔 50ms）
- **無障礙輔助** — 完整語音導覽（24 個 MP3 檔案）涵蓋所有遊戲狀態、玩家播報與操作說明

## 主控端 vs 從屬端韌體

主控端 (ESP32) 與從屬端搖桿 (ESP8266) 共用 `Protocol.h` 來定義封包格式與命令。兩者各自擁有依據自身角色量身打造的 `GameTypes.h`：

| | 主控端 (ESP32) | 從屬端 (ESP8266) |
|---|---|---|
| **角色** | 遊戲統籌、LED/音訊輸出 | 輸入裝置、發送結果 |
| **時脈** | 240 MHz | 160 MHz |
| **Protocol.h** | 包含顯示命令 (`DISP_*`) | 僅核心命令 |
| **NeoPixel 模式** | 8 種模式（rainbow、status、blink slot、shake countdown 等） | 6 種模式（無 blink slot / shake countdown） |
| **燈環對應** | 反向：P1=Ring4, P2=Ring3, P3=Ring1, P4=Ring0 | 循序：P1=Ring0, P2=Ring1, center=Ring2, P3=Ring3, P4=Ring4 |
| **狀態機** | 8 個遊戲狀態，具備完整回合管理 | 4 個搖桿狀態（idle、waiting GO、timing/shaking、done） |
| **音訊** | 非阻塞式 MP3 佇列，搭配 I2S + PWM 音量控制 | 無音訊硬體 |
| **週邊設備** | NeoPixel 燈環、WS2812B 燈條、I2S DAC、擴大機 | 按鈕、MPU-6050、震動馬達 |

### 搖桿狀態機

每支搖桿遵循以下每回合生命週期：

```
JS_IDLE ──> JS_WAITING_GO ──> JS_REACTION_TIMING  ──> JS_DONE
                          └──> JS_SHAKE_COUNTING   ──> JS_DONE
```

- **JS_IDLE** — 防彈跳按鈕輪詢；按下時發送帶有韌體版本的 `CMD_REQ_ID`
- **JS_WAITING_GO** — 已接收到 `CMD_GAME_START`；等待 `CMD_GO`
- **JS_REACTION_TIMING** — `micros()` 計時器執行中；按鈕中斷捕捉時間戳記
- **JS_SHAKE_COUNTING** — MPU-6050 高通濾波器計算完整推回週期；每 5 次搖晃回報進度
- **JS_DONE** — 結果已發送；等待下一回合或待機命令

## 相依套件

### 主控端 (ESP32)
| 函式庫 | 版本 | 用途 |
|---------|---------|---------|
| [NeoPixelBus](https://github.com/Makuna/NeoPixelBus) | ≥2.8.3 | LED 燈環 + 燈條控制（RMT DMA） |
| [ESP8266Audio](https://github.com/earlephilhower/ESP8266Audio) | 1.9.7 | MP3 解碼 + I2S 輸出 |

### 搖桿 (ESP8266)
| 函式庫 | 來源 | 用途 |
|---------|--------|---------|
| ESP8266WiFi | 內建 | ESP-NOW 傳輸 |
| Wire | 內建 | 用於 MPU-6050 的 I2C |

## 授權條款

MIT License

## 貢獻

歡迎各方貢獻！歡迎隨時提出 issue 或提交 pull request。
