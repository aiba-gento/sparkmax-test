# STM32G474RCT6によるSPARK MAX制御手順書（FDCAN3使用）

STM32G474RCT6のFDCAN3周辺機能を使用し、REV Robotics製SPARK MAXモーターコントローラを直接制御するための技術手順を以下に記す。

---

## 1. ハードウェアの接続と物理仕様

* **物理層の整合性**: STM32のFDCAN3_TX/RXピンをCANトランシーバーに接続し、SPARK MAXのCAN_H/L（緑・黄）ラインと結線する。
* **終端抵抗**: バス両端に120Ωの抵抗を配置する。
* **NEO 550 モーター仕様**:
* **Nominal Voltage**: 12 V。
* **Motor Kv**: 917 Kv。
* **Free Speed**: 11000 RPM。
* **Encoder Resolution**: 42 counts per rev。



## 2. 周辺機能の構成（CubeMX/CubeIDE設定）

FDCAN3に以下のパラメーターを設定する。

### 2.1 基本パラメーター

* **Frame Format**: `Classic mode` を選択する。
* **Mode**: `Normal mode` とする。
* **Auto Retransmission**: `Enable` に設定する。

### 2.2 ビットタイミング設定（1Mbps）

* **Nominal Prescaler**: `8`。
* **Nominal Time Seg1**: `16`。
* **Nominal Time Seg2**: `4`。
* **サンプリングポイント**: 約81.0%。

## 3. デバイス設定（SPARK MAX Configuration）

NEO 550を使用する場合、以下の設定を保存する必要がある。

### 3.1 モーター・物理パラメーター

* **Motor Type**: `BRUSHLESS` に設定する。
* **Idle Mode**: `BRAKE` または `COAST` を選択する。
* **Motor Kv**: `917` RPM/V に設定する。
* **Pole Pairs**: `7` に設定する。

### 3.2 電流制限（保護設定）

* **Smart Current Stall Limit**: `20 A`（NEO 550の焼損防止のため）。
* **Smart Current Free Limit**: `10 A - 15 A` を推奨。

## 4. エンコーダの選択と特性

制御目的に応じてフィードバック源を使い分ける。

* **Primary Encoder**:
* モーター内部のホールセンサーを使用する。
* 解像度は 42 counts/rev 固定。
* 転流（Commutation）および基本的な速度制御に使用される。


* **Alternate Encoder**:
* Data Portを介して外部センサーを接続する。
* ギアボックスの遊び（バックラッシュ）を排除した正確な位置制御に使用される。



## 5. CAN IDの構築アルゴリズム

SPARK MAXは29ビット拡張識別子を使用する。

| ビット範囲 | フィールド名 | 設定値 |
| --- | --- | --- |
| 24 - 28 | Device Type | `2` (Motor Controller) |
| 16 - 23 | Manufacturer | `5` (REV Robotics) |
| 10 - 15 | API Class | `0` (Control) または `6` (Status) |
| 6 - 9 | API Index | `1` (Duty Cycle) 等 |
| 0 - 5 | Device ID | 設定したID (1-62) |

## 6. 制御実装手順

* **送信（Heartbeat/Duty Cycle）**: API Index `1` を使用し、20ms周期で `-1.0` 〜 `1.0` の32bit Floatデータを送信する。
* **受信（Status監視）**: Status 0（電圧・フォルト）やStatus 1（電流・速度）を監視する。
* **異常対応**: `Gate Driver Fault` 等のフォルトが記録されている場合は、送信前にリセットコマンドを送るか、ハードウェア接続を再確認する。

## 7. 安全対策

* **タイムアウト**: 100ms以上ハートビートが途絶えると自動停止する仕様に留意する。
* **電圧監視**: Brownout Warning を防ぐため、電源供給能力を確保する。