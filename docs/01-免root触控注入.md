# 01 · 免 root 触控注入

## 1. 为什么不用常规办法

| 办法 | 为什么不灵 |
| --- | --- |
| `adb shell input tap` | 单指，且每次要新起一个 `app_process`，60~80ms 延迟；连打用的音游必崩 |
| `sendevent` / `/dev/input/event*` | 现代机型 SELinux 直接拦，root 才能写 |
| Accessibility 手势 | 只有一根手指，且系统会抢焦点 |
| root + `uinput` | 需要 root |

最后用的是 **scrcpy-server 的 UHID 支持**：它跑在 `shell` 身份（免 root），
通过 `/dev/uhid` 注册一块 HID 触摸屏。内核把它当**真实外接数位屏**挂到
`/dev/input`，游戏侧完全分不出和真手指的区别。

```
app ──(local socket)── adb ── forward tcp:xxxx localabstract:scrcpy ── scrcpy-server
                                                                       ├─ video  (H.264)
                                                                       └─ control(UHID_CREATE / INPUT / DESTROY)
```

手机版更进一步：**不用 adb 对端**，App 自己起一个 `adb` 子进程连自己的无线调试端口，
再 `forward` 到 scrcpy-server —— 于是"手机自己给自己当电脑"。

## 2. 控制通道报文

scrcpy 协议的 control socket 上，UHID 相关只有三条消息，都是大端：

```
[12] + u16 uid + u16 0 + u16 0 + u8 nameLen + name + u16 descLen + descriptor   # UHID_CREATE
[13] + u16 uid + u16 reportLen + report                                          # UHID_INPUT
[14] + u16 uid                                                                   # UHID_DESTROY
```

**描述符**：`Usage Page = Digitizer (0x0D)`，`Usage = Touch Screen (0x04)`，
然后复制 10 份 finger collection（每份声明 `Tip Switch`、`In Range`、
`Contact Identifier`、`X`、`Y`）。X/Y 的 `Logical Max` 填**面板自然方向的宽高**，
所以描述符必须等拿到真实分辨率之后再生成。

**报告固定 50 字节 = 10 指 × 5 字节**：

```
byte 0      : 状态（bit4 Tip Switch | bit5 In Range，即 0x30 表示按下）
byte 1..2   : u16le X
byte 3..4   : u16le Y
```

没按的手指也要发全 0，**整个 50 字节一次发完**，不能只发变化的那几根。

## 3. 唯一真正会翻车的细节：触点身份

HID 规范里，一根手指跨多个报告是靠 **Contact Identifier 不变**来识别的。
如果每次抬起都换 id、或者复用同一个 id 而不清零，内核不会产生
`ABS_MT_TRACKING_ID = -1` —— 系统眼里那根手指**永远按着**。

症状极有迷惑性：**整首歌只有第一个音符能判定**，后面全部 miss，
但日志上动作一条不少、坐标全对。

做法：**每根手指占固定 slot，Contact Identifier 恒等于 slot 号**；
抬手时只把 `Tip Switch` / `In Range` 清零，id 原样保留。
`slot` 同时被当作排程里的手指编号，所以"哪根手指在哪"在整局里是稳定的。

## 4. 坐标换向

描述符里写的是**面板自然方向**（通常竖屏），而游戏是横屏逻辑坐标。
App 读到 `rotation` 的时刻往往还是竖屏，所以换向要**支持会话中途改**：

| rotation | HID 坐标 |
| --- | --- |
| 0 | `(x, y)` |
| 1 | `(panelW − y, x)` |
| 2 | `(panelW − x, panelH − y)` |
| 3 | `(y, panelH − x)` |

换向后再 `clamp` 到 `[0, panel−1]`，否则 HID 描述符的逻辑范围会把越界坐标截掉，
表现为"手指永远贴着屏幕右边缘"。

## 5. 无线调试：配对码是唯一需要人做一次的步骤

免 root 的代价是 **`adb` 必须先配对**。Android 11+ 的"无线调试"要求 6 位配对码，
而系统那个"使用配对码配对设备"对话框**一离开设置就消失**（配对服务也跟着停）。
所以必须有个东西在对话框显示期间把码读出来，替用户去 `adb pair`。

**做法**：一个只读的 `AccessibilityService`（不是输入法、不是悬浮窗）：

* 只在**前台是系统设置**时扫描窗口文本；
* 只有当同一棵无障碍树里**同时存在**"配对码"字样**和**一个孤立的 6 位数字时才动作
  （"输入配对码配对设备"这一行本身带"配对码"三个字，很容易误命中，
  所以还要求出现真正的对话框标题：AOSP 是 `WLAN 配对码` / `Wi-Fi pairing code`）；
* 同时抓对话框里的 `192.168.x.x:port`；
* 不联网、不落盘、不记录其他内容。

**为什么不用悬浮键盘覆盖在设置页输入**：小米的设置页给窗口打了
`mIsForceHiddenNonSystemOverlayWindow = true`，任何权限都绕不过，浮窗在设置页上
一定是隐藏的。这条路走不通，只能读、不能画。

`adb pair <ip:port> <code>` 成功后，`adb connect <ip>:<port>`（**端口不同**：
配对端口是随机的，连接端口是无线调试页上显示的那个），之后就能 `forward` 了。

## 6. 没有 `/dev/uhid` 的机型

自动退回 scrcpy 的 `INJECT_TOUCH_EVENT`（走 `MotionEvent` 注入）。
能用，但它是**单指 + 事件合并**，多押会互相顶掉，只作为兜底。
PC 版和手机版都用同一套"先试 UHID、失败再退"的逻辑。