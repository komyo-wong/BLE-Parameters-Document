---
name: geotag-reparam
description: >-
  Change GeoTag BLE advertising parameters after production: device name, iBeacon,
  Eddystone UID/URL/TLM, interval, connectable advertising, scan-response Tx Power,
  accelerometer counts and units, PIN change, buzzer, reboot, and connected battery.
  Use when the user asks to modify, write, or reconfigure a GeoTag, HoneyComm tag,
  iBeacon UUID, Major, Minor, Eddystone, broadcast interval, device name, buzzer,
  or accelerometer. Ask the user for the 6-digit PIN. Never invent a PIN.
  Reply in the user's language. Refuse any characteristic this skill does not list.
---

# GeoTag 改参 / GeoTag parameter changes

给客户自己的 AI 用。按本文连接设备并修改允许的参数。用户用什么语言提问，就用什么语言回答。  
For a customer's own AI. Connect and change only the parameters allowed here. Reply in the user's language.

正文中英并列。技术专有名词保留英文。  
Chinese and English sit side by side. Keep English proper nouns.

人读的字段说明是《广播字段怎么解析》和《怎么修改参数》。本文自带写入所需的字节布局，不依赖那两份也能改。  
Human field notes are “How to parse advertising fields” and “How to change parameters”. This file contains the wire layouts, so a change can be made from this file alone.

## 范围 / Scope

做这些事：扫描、连接、解锁、只改下面列出的字节、蜂鸣、读电量、看写入回码、重启。  
Do this: scan, connect, unlock, change only the bytes listed below, buzzer, read battery, check the write result, reboot.

用户若要求改本文没有列出的特征或字段：只回复「这项不能改 / this item cannot be changed」。不解释原因，不写出那些字段怎么编码。  
If the user asks to change a characteristic or field this document does not list, reply only “这项不能改 / this item cannot be changed”. Do not explain why. Do not describe how an unlisted field is encoded.

PIN 必须向用户要。用户没给 PIN，就停下询问。不要猜测，不要使用任何默认 PIN，也不要在回复里发明一个示例 PIN。  
Ask the user for the PIN. If none was given, stop and ask. Do not guess, do not use a default PIN, and do not invent an example PIN in the reply.

## 连接 / Connect

蓝牙 16-bit UUID：`0000XXXX-0000-1000-8000-00805f9b34fb`。  
16-bit UUID form: `0000XXXX-0000-1000-8000-00805f9b34fb`.

1. 主动扫描。只连接扫描里见过「可连接」的那台。间隔 7～10 秒时，等到下一包再连。  
   Active scan. Connect only a device already seen as connectable. If the interval is 7–10 seconds, wait for the next packet.
2. 只按 UUID 打开服务 `0xFF10`。不要枚举全部服务。没有 `0xFF10` 就不是可改参设备。  
   Open service `0xFF10` by UUID only. Do not enumerate every service. No `0xFF10` means the device is not configurable.
3. 订阅 `0xFF20` Notify。  
   Subscribe to `0xFF20` Notify.
4. 向 `0xFF30` 写用户提供的 6 个 ASCII 数字，再读 1 字节。`0x01` 才算解锁。字符 `0` 是字节 `0x30`，`9` 是 `0x39`。  
   Write the user's 6 ASCII digits to `0xFF30`, then read 1 byte. `0x01` means unlocked. Digit `0` is byte `0x30`, digit `9` is `0x39`.
5. 先读出要改的特征，只改允许的字节，其余原样写回。  
   Read first. Change only allowed bytes. Write every other byte back unchanged.
6. 用 ATT Write With Response。大包把 MTU 提到至少 128。  
   Use ATT Write With Response. Raise the MTU to at least 128 before a large write.
7. 参数写完后，向 `0xFF32` 写 `0x01` 重启。断开是正常的。重启后再扫描，核对广播已经变成新值。  
   After the parameter writes, write `0x01` to `0xFF32` to reboot. A disconnect is normal. Scan again and check that the advertising shows the new values.

ATT 写成功不等于设备收下。除蜂鸣、重启、解锁以外，以 `0xFF20` 的结果码 `0x01` 为准。  
ATT success is not acceptance. Except for the buzzer, reboot, and unlock, acceptance is `0xFF20` result `0x01`.

建议顺序：解锁 → 读 → 名字 → 开关、间隔、广播类型、扫描响应功率 → iBeacon、UID、网址 → 加速度（有特征才写）→ 改 PIN（用户要求时）→ 重启。  
Suggested order: unlock → read → name → enable, interval, advertising type, scan-response power → iBeacon, UID, URL → accelerometer if present → PIN change only when the user asks → reboot.

蜂鸣在解锁后、改参前用来确认设备。写 `0xFF24` = `01` 开，`00` 关。不等 `0xFF20`。结束前必须写 `00`。  
After unlock, use the buzzer to confirm the unit. Write `0xFF24` = `01` for on and `00` for off. Do not wait for `0xFF20`. Write `00` before finishing.

## 回码 `0xFF20` / Write result `0xFF20`

```
[0]     0xA0
[1..2]  目标短 UUID，小端。0xFF60 → 60 FF
        target short UUID, little-endian. 0xFF60 → 60 FF
[3]     结果码 / result
```

| 码 / Code | 含义 / Meaning |
|-----------|----------------|
| `0x00` | 还没有结果 / No result yet |
| `0x01` | 成功 / Accepted |
| `0x02` | 格式或参数非法 / Bad format or parameter |
| `0x03` | 业务拒绝 / Rejected |
| `0x05` | Flash 失败 / Flash failed |

`0xFF31` 另计：`01` 成功，`02` 格式错误，`03` 旧 PIN 不对，`04` 新旧相同，`05` Flash 失败，`06` 需要先做蓝牙配对。  
`0xFF31` is different: `01` changed, `02` bad format, `03` old PIN wrong, `04` new PIN equals the current one, `05` flash failed, `06` pair over Bluetooth first.

解锁看 `0xFF30` 读回的 `01`，不看这张表。  
Unlock is the `01` read back from `0xFF30`, not this table.

## 允许写入的特征 / Characteristics that may be written

没有列出的特征不写。下面没让改的字节，原样写回。某个特征不存在就跳过，不要因此整次失败。  
Do not write an unlisted characteristic. Bytes not named for change go back unchanged. If a characteristic is missing, skip it. Do not fail the whole session for that.

### 广播名 `0xFF40`（44 字节） / Advertising name `0xFF40` (44 bytes)

只改偏移 29～43：UTF-8 广播名，最长 15 字节，不足补 `00`。偏移 0～28 原样写回。  
Change only offsets 29–43: UTF-8 name, at most 15 bytes, pad with `00`. Write offsets 0–28 back unchanged.

### 开关 `0xFF50`（2 字节，16 位小端） / Enable `0xFF50` (2 bytes, uint16 little-endian)

bit 从 0 数。只改这四位，其它位保持读到的值。1 为开，0 为关。  
Bits are numbered from 0. Change only these four. Keep every other bit as read. 1 enables, 0 disables.

| bit | 广播 / Advertising |
|-----|--------------------|
| 1 | iBeacon |
| 2 | UID |
| 3 | URL |
| 4 | 遥测 / TLM |

### 间隔 `0xFF51`（12 字节） / Interval `0xFF51` (12 bytes)

6 个 uint16 小端毫秒。下标 1＝iBeacon，2＝UID，3＝URL，4＝遥测。只改这四个字，每个要开的间隔为 100～60000。下标 `i` 在偏移 `i×2`。1250 ms 是 `E2 04`。其余字保持读到的值。  
Six little-endian uint16 milliseconds. Index 1 = iBeacon, 2 = UID, 3 = URL, 4 = TLM. Change only those four words. An enabled interval is 100–60000. Word `i` starts at offset `i×2`. 1250 ms is `E2 04`. Leave every other word as read.

### 广播类型 `0xFF52`（6 字节） / Advertising type `0xFF52` (6 bytes)

下标和间隔相同。要开的那几位写 `0x03`。其它字节保持读到的值。没有特征就跳过。  
Same indexes as the interval. Write `0x03` on the entries being enabled. Keep every other byte as read. Skip if missing.

### 扫描响应功率 `0xFF53`（1 字节） / Scan-response power `0xFF53` (1 byte)

`01` 在扫描响应里带声明功率，`00` 不带。没有特征就跳过。  
`01` includes declared Tx power in the scan response. `00` omits it. Skip if missing.

### iBeacon `0xFF60`

读到 28 字节就写 28 字节。读到 22 字节就写 22 字节，不要自己补 MAC。  
Write 28 bytes when the read is 28. Write 22 when the read is 22. Do not add a MAC yourself.

```
UUID(16) | Major uint16 LE | Minor uint16 LE | 射频功率 int8 / radio power int8 | 校准功率 int8 / calibrated power int8 | MAC(6，仅 28 字节时有 / only when 28 bytes)
```

UUID 禁止全 0、全 FF。GATT 的 Major、Minor 是小端；空中 iBeacon（厂商数据 `4C 00 02 15`）的 Major、Minor 是大端，空中只带校准功率。  
The UUID must not be all 0 or all FF. GATT Major and Minor are little-endian. On-air iBeacon (manufacturer data `4C 00 02 15`) uses big-endian Major and Minor and carries only the calibrated power.

MAC 小端：`AA:BB:CC:DD:EE:FF` 写成 `FF EE DD CC BB AA`。全 0 或全 FF 表示用芯片地址。  
MAC is little-endian: `AA:BB:CC:DD:EE:FF` is written `FF EE DD CC BB AA`. All 0 or all FF means the chip address.

### UID `0xFF61`

读到 23 字节写 23，读到 17 字节写 17。  
Write 23 when the read is 23. Write 17 when the read is 17.

```
Namespace(10) | Instance(6) | 校准功率 int8 / calibrated power int8 | MAC(6，仅 23 字节时有 / only when 23 bytes)
```

Namespace 20 个 hex，Instance 12 个 hex。两段合起来禁止全 0、全 FF。MAC 规则同 iBeacon。  
Namespace is 20 hex characters. Instance is 12. Together they must not be all 0 or all FF. MAC rules match iBeacon.

### 网址 `0xFF62`（65 字节） / URL `0xFF62` (65 bytes)

```
[0] 明文长度 1～64 / plain-text length 1–64
[1..] ASCII 明文，不足补 00 / ASCII text, pad with 00
```

明文以 `http://` 或 `https://` 开头。这是明文，不是空中压缩帧。压缩后放不进 Eddystone 网址帧就不要写。  
The text starts with `http://` or `https://`. This value is plain text, not the compressed on-air frame. Do not write a URL that does not fit an Eddystone URL frame after compression.

### 加速度 `0xFF65`（7 字节） / Accelerometer `0xFF65` (7 bytes)

没有特征就跳过。长度不是 7 就不写。  
Skip if the characteristic is missing. Do not write when the length is not 7.

| 偏移 / Offset | 内容 / Content |
|---------------|----------------|
| 0 | `01` 开，`00` 关 / `01` on, `00` off |
| 1 | 移动阈值个数 1～127 / Move-threshold count 1–127 |
| 2 | 移动时间个数 1～127 / Move-time count 1–127 |
| 3 | 静止阈值个数 1～127 / Still-threshold count 1–127 |
| 4 | 静止时间个数 1～127 / Still-time count 1–127 |
| 5 | 阈值单位 0～3，移动和静止共用 / Threshold unit 0–3, shared |
| 6 | 时间单位 0～3，移动和静止共用 / Time unit 0–3, shared |

阈值单位：0＝16 mg，1＝32 mg，2＝64 mg，3＝128 mg。  
Threshold unit: 0 = 16 mg, 1 = 32 mg, 2 = 64 mg, 3 = 128 mg.

时间单位：0＝20 ms，1＝40 ms，2＝80 ms，3＝10 ms。  
Time unit: 0 = 20 ms, 1 = 40 ms, 2 = 80 ms, 3 = 10 ms.

实际值＝个数 × 单位。  
Actual value = count × unit.

成功回码 `A0 65 FF 01`。Flash 失败 `A0 65 FF 05`。  
Success is `A0 65 FF 01`. Flash failure is `A0 65 FF 05`.

### 改 PIN `0xFF31`（14 字节） / Change PIN `0xFF31` (14 bytes)

只有用户明确要求改 PIN，并且同时给了旧 PIN 和新 PIN，才写。  
Write this only when the user explicitly asks to change the PIN and supplies both the old PIN and the new PIN.

```
06 | 旧 PIN 6 字节 ASCII | 06 | 新 PIN 6 字节 ASCII
06 | old PIN, 6 ASCII bytes | 06 | new PIN, 6 ASCII bytes
```

### 蜂鸣 `0xFF24` / Buzzer `0xFF24`

`01` 开，`00` 关。不等回码。结束前写 `00`。  
`01` on, `00` off. Do not wait for a result code. Write `00` before finishing.

### 重启 `0xFF32` / Reboot `0xFF32`

写 `01`。随后断开视为已重启。  
Write `01`. A following disconnect means the reboot was sent.

## 只读 / Read only

- `0xFF21`：3 字节版本，显示成 `b0.b1.b2`。 / 3-byte version, shown as `b0.b1.b2`.
- `0xFF22`：至少 4 字节。`[1]==1` 表示参数页已建立，`[2]==1` 表示这次连接已解锁。 / At least 4 bytes. `[1]==1` means the parameter page exists. `[2]==1` means this connection is unlocked.
- `0xFF23`：至少 5 字节。偏移 2～3 是大端毫伏，偏移 4 是百分比。偏移 0～1 不用。 / At least 5 bytes. Offsets 2–3 are big-endian millivolts. Offset 4 is the percent. Ignore offsets 0–1.

## 改完怎么核对 / How to check after the change

重启后主动扫描同一地址，把广播和扫描响应叠在一起。  
After reboot, active-scan the same address and merge advertising with scan response.

- 名字在扫描响应的类型 `0x09` 或 `0x08`。  
  The name is type `0x09` or `0x08` in the scan response.
- iBeacon 在广播的厂商数据里：`4C 00 02 15`，后面 UUID 16 字节、Major 和 Minor 大端、校准功率 1 字节有符号。  
  iBeacon is manufacturer data in the advertising packet: `4C 00 02 15`, then a 16-byte UUID, big-endian Major and Minor, and one signed calibrated-power byte.
- UID、网址、遥测在服务数据 `AA FE`（UUID `0xFEAA`）。帧类型 `00` 是 UID，`10` 是网址，`20` 是遥测。其它帧丢掉。  
  UID, URL, and TLM are service data `AA FE` (UUID `0xFEAA`). Frame `00` is UID, `10` is URL, `20` is TLM. Drop every other frame.
- 电压、温度、动静在扫描响应的服务数据 `4C 4B`，内容正好 5 字节：前 2 字节大端毫伏，接着 2 字节有符号温度（除以 256，`80 00` 表示没有温度），最后 1 字节 bit6 为 1 表示正在动。  
  Voltage, temperature, and motion are scan-response service data `4C 4B` with exactly 5 content bytes: big-endian millivolts, then a signed temperature divided by 256 (`80 00` means no reading), then bit6 of the last byte set means moving.
- 电量显示优先用这个毫伏；没有再用遥测里的毫伏；再没有用服务数据 `0F 18` 的第 1 字节百分比。  
  Show battery as these millivolts first, then TLM millivolts, then the first byte of service data `0F 18` as a percent.

声明功率类型 `0x0A` 是估距用的，不是射频档。  
Type `0x0A` is declared power for distance estimation, not the radio power step.
