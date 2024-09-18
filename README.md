# STM32duino-bootloader

请注意：此代码不适用于所有 STM32F103 主板

另请注意：请使用 GCC 4.8（不要使用 4.9 或更新版本，因为这些版本具有更强的优化功能，会导致硬件寄存器读取错误，从而导致引导加载程序无法工作）


适用于 STM32F103 板的引导加载程序，与 Arduino_STM32 repo 和 Arduino IDE 一起使用

该 repo 是 https://github.com/jonatanolofsson/maple-bootloader（mini-boot 分支）的派生版本，而 https://github.com/jonatanolofsson/maple-bootloader 又是由 Leaflabs http://github.com/leaflabs/maple-bootloader 创建的 maple-bootloader 的派生版本。

引导加载器经过重新设计，所有版本都包含在主分支中，而不是每个版本/板都有一个分支。

注意。
上传到 RAM 选项已被删除，因为它毫无用处，还会造成问题。
引导加载程序仍有一个用于 RAM 上传的 DFU - AltID，但如果主机试图上传到 AltID，则会返回错误。保留 AltID 是为了向后兼容仍在使用旧版 Maple IDE（该 IDE 有上传至 RAM 选项）的用户。

现在的 makefile 有多个构建目标，包括 “maple-mini”（大致相当于旧的 mini-boot 分支）、maple-rev3 等。

此外，引导加载程序现在还能在 “通用 ”STM32F103 板上运行，这些板没有额外的 USB 复位硬件，而所有 Maple 和 Maple mini 板都有。

在 “通用 ”板上，USB 复位（强制主机重新登录）是通过将 USB D+ 线 (PA12) 重新配置为 GPIO 模式，并在短时间内将 PA12 驱动为低电平来触发的，然后再将引脚设置回 USB 操作模式。
这种 USB 复位方法由 @Victor_pv 编写。
注意：它不能保证在所有 “通用 ”STM32 板上都能工作，并且依赖于 PA12 有一个约 1.5k 的上拉电阻，不过大多数 “通用 ”板似乎都有这个电阻。
目前还不清楚这种重置 USB 总线的方法是否完全符合 USB 标准，但它似乎在所有 PC、Mac 和 Linux 系统上都能正常工作，而且似乎适用于业余爱好/非商业/非关键系统。


通用 “STM32F103 板有多个构建目标，因为每个供应商似乎都在不同的端口/针脚上安装了 ”LED"，引导加载程序会闪烁 LED 指示当前的操作/状态。

因此，如果你的电路板 PC13 引脚上有一个 LED，那么 makefile 目标就是 “generic-pc13”。在撰写本文时，以下是可用的通用目标（可能会添加更多目标，但本自述每次都不会更新，因此请查看 makefile 以查看最新的构建目标列表）

通用-pc13
通用-pg15
通用-pd2
通用-pd1
通用-pa1
通用-pb9

除 LED 外，构建目标的差异还包括
(a) 设备是否有 maple USB 复位硬件。
(b) “按钮 ”连接到哪个引脚。

注意：大多数 “通用 ”STM32F103 板只有复位按钮，而没有用户/测试按钮。因此，引导程序代码总是将按钮输入引脚配置为 PullDown。因此，如果 Button 引脚（默认为 PC14）上没有按钮，该引脚应保持低电平状态，引导程序将认为按钮没有被按下。

重要！
如果电路板的 PC12 引脚连接了外部硬件，并将该引脚拉高，则需要为电路板创建一个新的构建目标，为按钮使用不同的引脚，或者修改代码使其忽略按钮。

每个构建目标/电路板的配置在 config.h 中定义。

Makefile 将电路板名称设置为 #define，例如 #define TARGET_GENERIC_F103_PD2，config.h 包含每个电路板的配置定义块。
例如

```
#define HAS_MAPLE_HARDWARE 1
#define LED_BANK GPIOB
#define LED_PIN 1
#define LED_ON_STATE 1
/* 在 Mini 上，BUT 为 PB8 */
#define BUTTON_BANK GPIOB
#define BUTTON 8
/* USB 磁盘引脚设置。USB DISC 为 PB9 */
#define USB_DISC_BANK GPIOB
#define USB_DISC 9
#define USER_CODE_RAM ((u32)0x20000C00)
#define RAM_END ((u32)0x20005000)
```

带有 Maple USB 复位硬件的电路板需要定义 HAS_MAPLE_HARDWARE。

有 LED_BANK（GPIO 端口）的定义
LED 引脚（该端口的引脚）
LED_ON_STATE（当引脚为高电平（1）或低电平（0）时，LED 是否点亮--注意：这是因为某些通用电路板的 LED 位于 Vcc 和引脚之间，因此当 LED 引脚为低电平时，LED 点亮。）

按钮组（端口）和引脚

对于带 Maple 硬件的电路板，需要定义 “断开 ”引脚 USB_DISC。
在 “通用 ”电路板上，由于在 STM32F103 设备上 USB D+ 始终位于该引脚上，因此该引脚始终为 PA12。
(注：对于通用电路板，该定义可移至 usb 代码中，因为它不会发生变化。）

请注意：
USER_CODE_RAM 和 RAM_END 始终相同，可能需要合并，而不是为每块板定义。

不再需要配置闪存大小（FLASH SIZE），因为它是通过从处理器寄存器读取大小来确定的。
闪存页面（以及 DFU 上载块大小）也不再需要定义，因为闪存页面大小是根据闪存总大小推断出来的。所有 128kb 以下的设备都有 1k 的页面大小，而闪存超过 128k 的电路板则有 2k 的页面大小。

请注意，代码直接将闪存页面大小与 DFU 块大小联系起来。也许可以将其分离，从而提高上传速度，但目前尚未实现。


##### 对原始 Maple 引导加载程序的其他改进

1. 更小的占用空间 - 现在只有 8k（这是因为 @jonatanolofsson 修改了代码，从而可以使用 GCC 优化大小标志
2. 增加了额外的 DFU AltID 上传类型，允许在 0x8002000 而不是 0x8005000 加载草图（由于引导加载程序本身的大小减小）、
注：为实现向后兼容性，保留了上载到 0x8005000 的功能。


如果您对引导加载程序有任何疑问，请提出问题，我将尽力为您解答。




