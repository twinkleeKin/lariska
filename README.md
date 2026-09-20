# HXK修改版
适配新的ZMK，由于支持Zephyr4.1导致了设备树和板子名字出现变化，同时3365传感器的git库也有所变化，导致原来不可用错误太多。
此版本已经一一修复。
# ZMK config

ZMK config for reverse engineered VGN F1/VXE R1 mouse powered by nrf52840 and PAW3395 sensor. You can also find Adafruit bootloader for this mouse [here](https://github.com/greengrocer98/Adafruit_nRF52_Bootloader).

## Pinout

Full pinout of the mouse if you would like to add additional functionality

| left button       | P0.30 |
| ----------------- | ----- |
| right button      | P0.29 |
| middle button     | P0.28 |
| forward button    | P0.03 |
| back button       | P0.02 |
| LDO enable front💡| P1.02 |
| red front led     | P0.16 |
| green front led   | P0.15 |
| blue front led    | P0.14 |
| motion pin        | P0.07 |
| clock pin         | P0.06 |
| mosi              | P0.05 |
| miso              | P0.04 |
| cs pin            | P0.27 |
| enc A             | P0.00 |
| enc B             | P0.01 |
| voltage divider   | P0.31 |
| TP4056 CHRG       | P0.26 |
| DPI button        | P1.15 |
| LDO enable bottom💡| P1.01 |
| blue bottom led   | P0.21 |
| red bottom led    | P0.24 |
| green bottom led  | P0.22 |

💡 - LDO is used for powering LEDS. It should be enabled through enable pin.

## ESB peripheral 改造（配对 corne_esb_dongle 接收器）

这个分支的鼠标固件被改成了 **ESB split peripheral**：不再用 BLE 直连主机，而是把
指针/按键事件通过 ESB（2.4G 私有协议）发给接收器 `corne_esb_dongle`，由接收器统一
用 USB HID 与主机通信（键鼠套装）。

改了这些文件：

| 文件 | 改动 |
| --- | --- |
| `config/west.yml` | 增加 `zmk-feature-split-esb`（badjeff，`zmk-0.4`）、`sdk-nrf`（badjeff `v3.1-branch+zmk-fixes`，`path: nrf`）、`nrfxlib`（nrfconnect `v3.1-branch`） |
| `boards/shields/lariska/lariska.conf` | `CONFIG_ZMK_BLE=n`、`CONFIG_ZMK_SPLIT_ESB=y`、`ZMK_SPLIT_ESB_PERIPHERAL_ID=3`、ESB 参数（与接收器一致）、`CONFIG_ZMK_POINTING=y`、`CONFIG_ZMK_BATTERY_REPORTING=y`、`CONFIG_ZMK_SLEEP=n`。**5 个按键的配置一个字都没动** |
| `boards/shields/lariska/lariska.overlay` | 新增 `esb_split` 收发地址；新增 `split_inputs` 里的 `mou0_split@4`（指针 → input 通道）。**原来的 5 键 gpio-direct kscan 原样保留** |
| `boards/shields/lariska/lariska.keymap` | 原 `mou0_mmv_il`（input-listener，central 用法）改成 `status = "disabled"`；限速处理器挪到 `&mou0_split` 上。**按键绑定（`&mkp ...`）原样保留** |

要点：

- **鼠标的按键定义保持原样**：仍然是 gpio-direct kscan，上报 key position 0~4。
  与 Corne 键位撞车的问题是在**接收器侧**解决的 —— 接收器的
  `config/corne.overlay` 在 Corne 的 matrix-transform 前面插了 5 个占位条目，
  把 Corne 的 42 键整体挪到位置 5~46，把 0~4 让给鼠标；接收器的
  `config/corne.keymap` 再把位置 0~4 定义成 `&mkp LCLK/RCLK/MCLK` 与
  `u_lt_mouse 3 MB4` / `u_lt_mouse 4 MB5`（按住进层 3/4 = 音量 / 滚轮）。
- **ESB 地址必须与接收器的 `config/corne.overlay` 逐字节一致**，`PERIPHERAL_ID`
  不能与左右手（1、2）重复。
- 指针走 input 通道：`mou0_split`(reg=4) → 接收器的 `mouse_split` + `mouse_listener`。
- 滚轮走 sensor 事件：接收器侧有一个占位 `zmk,keymap-sensors`（挂在没用的
  P0.09/P0.10 上的 `alps,ec11`），配合 keymap 每层的 `sensor-bindings` 实现滚轮/音量。
- 按键唤醒/睡眠暂未处理（`CONFIG_ZMK_SLEEP=n`），链路验证通过后再做。

构建：`build.yaml` 里的 `nice_nano//zmk` + `lariska`，走 GitHub Actions 或本地 `west build`
都可以；需要跟接收器用同一个 ZMK commit（官方 `9ebbeff0`，已 pin 在 `config/west.yml`）。
