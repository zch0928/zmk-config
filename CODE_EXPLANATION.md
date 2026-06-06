# ZMK 配置代码阅读说明

这份文档用你熟悉的 Python、Java、HTML、Vue 视角解释当前仓库。这个项目不是传统的应用程序项目，而是一个 ZMK 键盘固件配置仓库：你写的是“声明式配置”，真正的 C 代码、驱动、蓝牙协议、USB 协议都由 ZMK 和 Zephyr 提供。

## 一句话理解

这个仓库描述了一块名叫 `macro8` 的 8 键宏键盘：

- 主控芯片是 Nordic `nRF52840`。
- 有 8 个直接接 GPIO 的按键。
- 支持 USB 和蓝牙。
- 有 1 颗 WS2812 RGB 灯。
- 编译产物希望输出 `.hex` 固件文件。
- 8 个按键里，前 4 个是文字宏，后 4 个是音量和输出切换。

可以把它类比成一个 Vue 项目：

- `ZMK/Zephyr` 像 Vue/Vite/浏览器运行时，是底层框架。
- `config/*.conf` 像 `.env` 或配置开关。
- `*.dts` 像用树形结构声明硬件 DOM。
- `*.keymap` 像声明“用户点击按钮时触发什么行为”。
- `build.yaml` 和 GitHub Actions 像 CI 构建配置。

## 仓库结构

```text
.
├── .github/workflows/build.yml
├── .gitattributes
├── .gitignore
├── build.yaml
├── boards/shields/.gitkeep
├── config/
│   ├── west.yml
│   ├── macro8.conf
│   ├── macro8.keymap
│   └── boards/arm/macro8/
│       ├── Kconfig.board
│       ├── Kconfig.defconfig
│       ├── macro8.dts
│       ├── macro8.yaml
│       └── macro8_defconfig
└── zephyr/module.yml
```

`.zmk/` 是本地拉取下来的 ZMK 上游源码目录，已经被 `.gitignore` 忽略。你平时主要维护上面列出的这些文件。

## 构建流程

从配置到固件大致是这样：

```text
GitHub Actions
  读取 .github/workflows/build.yml
    调用 ZMK 官方 build-user-config workflow
      读取 build.yaml，知道要构建 board: macro8
        通过 config/west.yml 拉取 ZMK 和 Zephyr
          通过 zephyr/module.yml 找到自定义 board 根目录
            读取 macro8 的硬件定义、功能开关、按键映射
              生成 firmware .hex
```

本地构建时流程类似，只是由 `west build` 触发，而不是 GitHub Actions 触发。

## 文件逐个解释

### `.github/workflows/build.yml`

```yaml
name: Build ZMK firmware
on: [push, pull_request, workflow_dispatch]

jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3
    with:
      fallback_binary: hex
```

这是 GitHub Actions 配置。

- `name`：工作流名称，在 GitHub Actions 页面显示。
- `on`：什么时候自动构建。
  - `push`：推送代码时构建。
  - `pull_request`：创建或更新 PR 时构建。
  - `workflow_dispatch`：允许你在 GitHub 页面手动点按钮构建。
- `jobs.build.uses`：不自己写构建步骤，而是复用 ZMK 官方的用户配置构建流程。
- `@v0.3`：使用 ZMK v0.3 版本的构建流程。
- `fallback_binary: hex`：当没有 `.uf2` 产物时，把 `.hex` 作为下载产物，而不是默认的 `.bin`。

如果你以后想升级 ZMK，需要同时考虑这里的 `@v0.3` 和 `config/west.yml` 里的 `revision: v0.3`。

### `build.yaml`

```yaml
---
include:
  - board: macro8
```

这是 ZMK 用户配置的构建矩阵。

当前只有一个构建目标：

- 板子名称：`macro8`

它会去找这个路径下的板子定义：

```text
config/boards/arm/macro8/
```

如果以后你有左右手分体键盘，或者多个板子，这里可以增加多个条目。比如：

```yaml
include:
  - board: macro8
  - board: another_board
```

### `config/west.yml`

```yaml
manifest:
  defaults:
    revision: v0.3
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
  projects:
    - name: zmk
      remote: zmkfirmware
      import: app/west.yml
  self:
    path: config
```

`west` 是 Zephyr 的项目管理工具，有点像嵌入式世界里的包管理器 + 多仓库管理器。

关键字段：

- `revision: v0.3`：指定使用 ZMK 的 `v0.3` 版本。
- `remotes`：声明 ZMK 仓库来自 GitHub 的 `zmkfirmware` 组织。
- `projects`：拉取 `zmk` 项目。
- `import: app/west.yml`：继续读取 ZMK 自己的依赖配置，例如 Zephyr、hal、nrfx 等。
- `self.path: config`：说明当前这个用户配置仓库在 west 工作区里叫 `config`。

类比 Java/Maven：

- `west.yml` 有点像 `pom.xml` 里的依赖声明。
- `revision` 像依赖版本号。

### `zephyr/module.yml`

```yaml
build:
  settings:
    board_root: .
```

这告诉 Zephyr：当前模块里有自定义 board，board 根目录就是当前仓库。

因此 Zephyr 会扫描：

```text
config/boards/arm/macro8/
```

如果没有这个文件，构建系统可能找不到你的 `macro8` 板子。

### `.gitignore`

```gitignore
.zmk/
```

忽略本地下载的 ZMK 源码目录。

`.zmk/` 很大，而且是依赖源码，不应该提交到你的配置仓库里。真正需要提交的是你的配置文件。

### `.gitattributes`

```gitattributes
* text=auto eol=lf
```

告诉 Git 自动处理文本文件换行，并统一用 LF 换行。

这对 Zephyr/ZMK 这类跨平台项目比较重要，可以减少 Windows 和 Linux 之间的换行差异。

### `boards/shields/.gitkeep`

这是一个空占位文件。

Git 默认不追踪空目录，所以放一个 `.gitkeep` 让 `boards/shields/` 目录保留下来。当前项目没有自定义 shield，所以这个目录只是预留。

## 键盘功能配置

### `config/macro8.conf`

这个文件是 ZMK 应用层配置，主要控制键盘功能。

可以把它理解成 `.env` 或 Spring Boot 的 `application.properties`，只是这里的写法是 Kconfig：

```conf
CONFIG_功能名=y
CONFIG_功能名=n
CONFIG_数值=123
CONFIG_字符串="xxx"
```

常见含义：

- `y`：启用。
- `n`：禁用。
- 数字：配置数值。
- 字符串：配置名称。

当前配置意图如下：

```conf
CONFIG_ZMK_KEYBOARD_NAME="MacroKey"
```

设置键盘名称。蓝牙搜索设备时通常会看到这个名字。

```conf
CONFIG_ZMK_USB=y
```

启用 USB 键盘功能。

```conf
CONFIG_ZMK_BLE=y
CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y
```

启用蓝牙功能，并启用一个实验性的蓝牙连接相关配置。

```conf
CONFIG_ZMK_RGB_UNDERGLOW=y
CONFIG_ZMK_RGB_UNDERGLOW_EXT_POWER=n
CONFIG_WS2812_STRIP=y
```

启用 RGB 底灯功能，灯珠类型是 WS2812。

```conf
CONFIG_ZMK_BATTERY_REPORTING=n
```

禁用电池电量上报。注释里提到原因是当前 ZMK v0.3 的电池分压读取代码和 Zephyr 4.1.0 附带的 nrfx 3.x API 不兼容。

```conf
CONFIG_ZMK_SLEEP=y
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=900000
```

启用睡眠省电。`900000` 通常按毫秒理解，也就是 15 分钟无操作后进入睡眠。

### `config/macro8.keymap`

这是按键映射文件，决定“每个按键按下后做什么”。

它使用的是 devicetree 风格语法，看起来像 C，但本质更像 HTML/Vue 模板：声明一棵树，每个节点有属性。

文件开头：

```dts
#include <behaviors.dtsi>
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/bt.h>
#include <dt-bindings/zmk/outputs.h>
```

这些类似 Java/Python 的 import：

- `behaviors.dtsi`：导入 ZMK 的行为定义，比如按键、宏、层切换。
- `keys.h`：导入键码，例如 `A`、`RET`、`SPACE`、`C_VOL_UP`。
- `bt.h`：导入蓝牙相关常量。
- `outputs.h`：导入输出切换相关常量。

#### 宏定义

```dts
macros {
    macro_claude: macro_claude {
        compatible = "zmk,behavior-macro";
        #binding-cells = <0>;
        bindings = <&kp C &kp L &kp A &kp U &kp D &kp E &kp RET>;
    };
};
```

这段定义了一个宏 `macro_claude`。

类比 Vue：

```js
const macro_claude = () => {
  press("C")
  press("L")
  press("A")
  press("U")
  press("D")
  press("E")
  press("Enter")
}
```

关键点：

- `macro_claude:` 是标签，后面按键映射里可以用 `&macro_claude` 引用它。
- `compatible = "zmk,behavior-macro"` 表示这是 ZMK 宏行为。
- `#binding-cells = <0>` 表示调用这个宏时不需要额外参数。
- `bindings = <...>` 是宏实际执行的一串动作。
- `&kp` 表示 key press，也就是按一个键。
- `RET` 是 Enter。
- `SPACE` 是空格。
- `LC(V)` 是 Left Control + V，也就是 Ctrl+V。

当前 4 个宏：

| 宏名 | 实际输入 |
| --- | --- |
| `macro_claude` | `CLAUDE` 后回车 |
| `macro_zhcn` | `Ctrl+V` 后回车 |
| `macro_git` | `GIT ADD .` 后回车 |
| `macro_npm` | `NPM RUN DEV` 后回车 |

注意：这里的字母键码是键盘按键，不是文本输入法。实际大小写可能受键盘布局、Shift、Caps Lock、输入法影响。`&kp G` 通常表示按下 G 键，不一定等价于输出大写 `G` 字符。

#### 按键布局

```dts
keymap {
    compatible = "zmk,keymap";

    default_layer {
        bindings = <
            &macro_claude  &macro_zhcn  &macro_git   &macro_npm
            &kp C_VOL_DN   &kp C_VOL_UP &kp C_MUTE   &out OUT_TOG
        >;
    };
};
```

这里定义默认层的 8 个按键。

因为 `macro8.dts` 里定义了 8 个 GPIO 输入，所以这里也需要 8 个 binding，顺序一一对应。

当前布局：

```text
第 1 行:
1: macro_claude
2: macro_zhcn
3: macro_git
4: macro_npm

第 2 行:
5: 音量减小
6: 音量增大
7: 静音
8: 输出切换 USB/BLE
```

其中：

- `&kp C_VOL_DN`：Consumer Volume Down，音量减。
- `&kp C_VOL_UP`：Consumer Volume Up，音量加。
- `&kp C_MUTE`：静音。
- `&out OUT_TOG`：切换输出目标，例如 USB 和 BLE。

## 自定义板子定义

目录：

```text
config/boards/arm/macro8/
```

这部分是最接近硬件和 C/嵌入式概念的地方。

### `macro8.yaml`

```yaml
file_format: "1"
id: macro8
name: Macro8
type: board
arch: arm
outputs:
  - usb
  - ble
```

这是板子的元数据。

- `id: macro8`：板子 ID，必须和 `build.yaml` 里的 `board: macro8` 对上。
- `name: Macro8`：人类可读名称。
- `type: board`：这是一个完整 board，不是 shield。
- `arch: arm`：CPU 架构是 ARM。
- `outputs`：支持 USB 和 BLE 输出。

### `Kconfig.board`

```kconfig
config BOARD_MACRO8
    bool "Macro8 ..."
    depends on SOC_NRF52840_QIAA
```

这个文件告诉 Zephyr/Kconfig：存在一个叫 `BOARD_MACRO8` 的板子配置选项。

关键点：

- `config BOARD_MACRO8`：定义一个配置项。
- `bool`：这是布尔开关。
- `depends on SOC_NRF52840_QIAA`：只有当芯片是 `nRF52840 QIAA` 时，这个板子选项才合法。

类比 Java：

```java
if (soc == NRF52840_QIAA) {
    enableBoard("MACRO8");
}
```

### `Kconfig.defconfig`

```kconfig
if BOARD_MACRO8
# Board defaults are defined in macro8_defconfig.
endif
```

这个文件通常放板子的默认配置。当前它只是说明默认配置在 `macro8_defconfig` 里。

### `macro8_defconfig`

这是板子的默认 Kconfig 配置。

#### 芯片和板子身份

```conf
CONFIG_SOC_SERIES_NRF52X=y
CONFIG_SOC_NRF52840_QIAA=y
CONFIG_BOARD_MACRO8=y
```

说明：

- 使用 Nordic nRF52 系列。
- 具体芯片是 `nRF52840_QIAA`。
- 当前板子是 `macro8`。

#### 固件产物格式

```conf
CONFIG_BUILD_OUTPUT_HEX=y
CONFIG_BUILD_OUTPUT_BIN=n
```

说明：

- 生成 Intel HEX 格式固件。
- 不生成默认 BIN 格式固件。

这和 `.github/workflows/build.yml` 里的 `fallback_binary: hex` 配合，确保 GitHub Actions 最终产物是 `.hex`。

#### 时钟

```conf
CONFIG_CLOCK_CONTROL_NRF_K32SRC_XTAL=y
CONFIG_CLOCK_CONTROL_NRF_K32SRC_20PPM=y
```

说明使用外部 32.768 kHz 晶振作为低频时钟源，精度约 20 ppm。

蓝牙低功耗设备通常需要稳定的低频时钟。

#### USB

```conf
CONFIG_USB_DEVICE_STACK=y
CONFIG_USB_DEVICE_HID=y
CONFIG_ZMK_USB=y
```

说明：

- 启用 USB 设备协议栈。
- 启用 USB HID，也就是键盘、鼠标、媒体键这类设备协议。
- 启用 ZMK 的 USB 输出。

#### 蓝牙

```conf
CONFIG_BT=y
CONFIG_BT_CTLR=y
CONFIG_BT_MAX_CONN=5
```

说明：

- 启用蓝牙。
- 启用蓝牙控制器。
- 最多支持 5 个蓝牙连接配置。

#### RGB

```conf
CONFIG_SPI=y
CONFIG_WS2812_STRIP=y
CONFIG_WS2812_STRIP_SPI=y
```

说明：

- 启用 SPI。
- 启用 WS2812 灯带驱动。
- 使用 SPI 方式驱动 WS2812。

#### ADC 和 GPIO

```conf
CONFIG_ADC=y
CONFIG_GPIO=y
```

说明：

- ADC 用于读取模拟电压，例如电池电压。
- GPIO 用于读取按键和控制引脚。

### `macro8.dts`

这是最重要的硬件描述文件。`.dts` 是 Device Tree Source，设备树源码。

你可以把它想成浏览器 DOM：

```html
<board>
  <gpio />
  <adc />
  <spi>
    <ws2812 />
  </spi>
</board>
```

区别是 `.dts` 描述的是硬件节点。

#### 文件头

```dts
/dts-v1/;
#include <nordic/nrf52840_qiaa.dtsi>
#include <zephyr/dt-bindings/pinctrl/nrf-pinctrl.h>
```

说明：

- `/dts-v1/;`：声明这是 DTS v1 格式。
- `nrf52840_qiaa.dtsi`：导入 nRF52840 芯片的基础硬件定义。
- `nrf-pinctrl.h`：导入 Nordic 引脚复用宏。

`.dtsi` 可以理解成可被 include 的 DTS 片段，类似 Vue 组件或 Java import。

#### 根节点

```dts
/ {
    model = "macro8";
    compatible = "zmk,macro8";
};
```

根节点代表整块板子。

- `model`：板子型号名。
- `compatible`：驱动匹配字符串，告诉系统这是什么类型的硬件。

#### chosen 节点

```dts
chosen {
    zephyr,code-partition = &code_partition;
    zephyr,sram = &sram0;
    zephyr,flash = &flash0;
    zmk,battery = &vbatt;
    zmk,kscan = &kscan0;
    zmk,underglow = &led_strip;
};
```

`chosen` 是“告诉系统默认使用哪个硬件节点”的地方。

类比 Vue 里把某个组件注册成默认实现：

```js
app.provide("kscan", kscan0)
app.provide("battery", vbatt)
app.provide("underglow", led_strip)
```

这里的 `&xxx` 是引用标签：

- `&code_partition`：固件代码放在哪个 flash 分区。
- `&sram0`：使用哪块 RAM。
- `&flash0`：使用哪块 Flash。
- `&vbatt`：电池电压读取节点。
- `&kscan0`：按键扫描节点。
- `&led_strip`：RGB 灯节点。

#### 按键扫描 `kscan0`

```dts
kscan0: kscan {
    compatible = "zmk,kscan-gpio-direct";
    label = "KSCAN";
    input-gpios
        = <&gpio1 11 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>
        , <&gpio1 10 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>
        , <&gpio0  3 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>
        , <&gpio1 13 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>
        , <&gpio0 13 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>
        , <&gpio0 22 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>
        , <&gpio0 24 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>
        , <&gpio1  0 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>
        ;
};
```

这是 8 个按键的硬件输入定义。

关键点：

- `kscan0:` 是标签，后面 `chosen` 里用 `&kscan0` 引用。
- `compatible = "zmk,kscan-gpio-direct"`：使用 GPIO 直连扫描方式。
- `input-gpios`：列出 8 个按键连接到哪些 GPIO。

每一项格式：

```dts
<&gpio端口 引脚号 标志>
```

例如：

```dts
<&gpio1 11 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>
```

意思是：

- 使用 GPIO1 端口。
- 引脚号是 11。
- 按下时为低电平。
- 启用内部上拉电阻。

`GPIO_ACTIVE_LOW | GPIO_PULL_UP` 常见于按键电路：

- 没按时：上拉到高电平。
- 按下时：接地，变成低电平。

这 8 个 GPIO 的顺序非常重要，它决定了 `macro8.keymap` 里 8 个 binding 的对应关系。

| 顺序 | GPIO | 当前 keymap 对应动作 |
| --- | --- | --- |
| 1 | `gpio1 pin 11` | `macro_claude` |
| 2 | `gpio1 pin 10` | `macro_zhcn` |
| 3 | `gpio0 pin 3` | `macro_git` |
| 4 | `gpio1 pin 13` | `macro_npm` |
| 5 | `gpio0 pin 13` | 音量减 |
| 6 | `gpio0 pin 22` | 音量加 |
| 7 | `gpio0 pin 24` | 静音 |
| 8 | `gpio1 pin 0` | 输出切换 |

#### 电池电压节点 `vbatt`

```dts
vbatt: vbatt {
    compatible = "zmk,battery-voltage-divider";
    label = "VBATT";
    io-channels = <&adc 7>;
    output-ohms = <100000>;
    full-ohms = <200000>;
};
```

这定义了电池电压读取方式。

- `compatible = "zmk,battery-voltage-divider"`：使用电阻分压读取电池电压。
- `io-channels = <&adc 7>`：使用 ADC 的第 7 通道。
- `output-ohms = <100000>`：下拉电阻是 100k。
- `full-ohms = <200000>`：总电阻是 200k。

也就是典型的两个 100k 电阻分压：

```text
电池正极
  |
  100k
  |
  ADC 输入点
  |
  100k
  |
GND
```

不过当前 `macro8.conf` 里禁用了电池上报，所以这个节点被定义了，但功能没有启用。

#### USB 设备

```dts
&usbd {
    status = "okay";
    cdc_acm_uart0: cdc_acm_uart0 {
        compatible = "zephyr,cdc-acm-uart";
    };
};
```

`&usbd` 引用芯片自带的 USB 设备控制器，并启用它。

- `status = "okay"`：启用这个硬件节点。
- `cdc_acm_uart0`：定义一个 USB 虚拟串口。

键盘 HID 主要由 ZMK 配置启用；这里额外有串口节点，通常用于日志或调试。

#### ADC 和 GPIO

```dts
&adc {
    status = "okay";
};

&gpio0 {
    status = "okay";
};

&gpio1 {
    status = "okay";
};
```

启用 ADC、GPIO0、GPIO1。

如果某个硬件节点没有 `status = "okay"`，Zephyr 可能不会启用对应驱动。

#### SPI 引脚复用

```dts
&pinctrl {
    spi1_default: spi1_default {
        group1 {
            psels = <NRF_PSEL(SPIM_MOSI, 0, 26)>;
        };
    };

    spi1_sleep: spi1_sleep {
        group1 {
            psels = <NRF_PSEL(SPIM_MOSI, 0, 26)>;
            low-power-enable;
        };
    };
};
```

这段定义 SPI1 的 MOSI 引脚。

- `NRF_PSEL(SPIM_MOSI, 0, 26)`：把 `P0.26` 配置为 SPI MOSI。
- `spi1_default`：正常工作时的引脚配置。
- `spi1_sleep`：睡眠时的低功耗引脚配置。

WS2812 只需要单线数据，所以这里只配置了 MOSI。

#### SPI1 和 WS2812 灯

```dts
&spi1 {
    compatible = "nordic,nrf-spim";
    status = "okay";
    pinctrl-0 = <&spi1_default>;
    pinctrl-1 = <&spi1_sleep>;
    pinctrl-names = "default", "sleep";

    led_strip: ws2812@0 {
        compatible = "worldsemi,ws2812-spi";
        label = "WS2812";
        reg = <0>;
        spi-max-frequency = <4000000>;
        chain-length = <1>;
        spi-one-frame  = <0x70>;
        spi-zero-frame = <0x40>;
        color-mapping  = <0 1 2>;
    };
};
```

这段启用 SPI1，并在 SPI1 总线上挂了一个 WS2812 灯带设备。

关键点：

- `&spi1`：引用芯片内部的 SPI1 控制器。
- `compatible = "nordic,nrf-spim"`：使用 Nordic 的 SPI master 驱动。
- `pinctrl-0` / `pinctrl-1`：绑定默认和睡眠两套引脚配置。
- `led_strip:`：给 WS2812 节点起标签，`chosen` 里用 `&led_strip` 引用。
- `ws2812@0`：SPI 设备地址是 0。
- `spi-max-frequency = <4000000>`：SPI 最大频率 4 MHz。
- `chain-length = <1>`：只有 1 颗灯。
- `color-mapping = <0 1 2>`：颜色顺序，通常表示 RGB。

如果以后你接了 8 颗 WS2812，就把：

```dts
chain-length = <1>;
```

改成：

```dts
chain-length = <8>;
```

前提是供电和硬件连接也正确。

#### Flash 分区

```dts
&flash0 {
    partitions {
        compatible = "fixed-partitions";
        #address-cells = <1>;
        #size-cells    = <1>;

        sd_partition: partition@0 {
            label = "mbr";
            reg   = <0x00000000 0x00001000>;
        };
        code_partition: partition@1000 {
            label = "code";
            reg   = <0x00001000 0x000d3000>;
        };
        storage_partition: partition@d4000 {
            label = "storage";
            reg   = <0x000d4000 0x00020000>;
        };
        boot_partition: partition@f4000 {
            label = "adafruit_boot";
            reg   = <0x000f4000 0x0000c000>;
        };
    };
};
```

这定义 nRF52840 的 Flash 怎么分区。

`reg` 的格式：

```dts
reg = <起始地址 大小>;
```

当前分区：

| 分区 | 起始地址 | 大小 | 用途 |
| --- | --- | --- | --- |
| `mbr` | `0x00000000` | `0x00001000` | MBR/启动相关 |
| `code` | `0x00001000` | `0x000d3000` | ZMK 固件代码 |
| `storage` | `0x000d4000` | `0x00020000` | 存储配置、蓝牙配对等 |
| `adafruit_boot` | `0x000f4000` | `0x0000c000` | Adafruit bootloader |

`chosen` 里这行：

```dts
zephyr,code-partition = &code_partition;
```

告诉 Zephyr：真正的固件代码应该放进 `code_partition`。

这部分不要随便改。Flash 分区错了可能导致固件无法启动，或者覆盖 bootloader、存储区。

## DTS 语法速查

### 节点

```dts
node_name {
    property = value;
};
```

类似 HTML 标签或 JS 对象。

### 标签

```dts
label_name: node_name {
};
```

`label_name` 可以被其他地方用 `&label_name` 引用。

### 引用

```dts
zmk,kscan = &kscan0;
```

`&kscan0` 表示引用名为 `kscan0` 的节点。

### 数字数组

```dts
reg = <0x00001000 0x000d3000>;
```

尖括号里是整数数组。

### 字符串

```dts
label = "WS2812";
```

普通字符串。

### 多个值

```dts
pinctrl-names = "default", "sleep";
```

多个字符串用逗号分隔。

### 状态

```dts
status = "okay";
```

`okay` 表示启用。常见禁用写法是：

```dts
status = "disabled";
```

## Kconfig 语法速查

### 启用功能

```conf
CONFIG_ZMK_USB=y
```

### 禁用功能

```conf
CONFIG_ZMK_BATTERY_REPORTING=n
```

### 设置数值

```conf
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=900000
```

### 设置字符串

```conf
CONFIG_ZMK_KEYBOARD_NAME="MacroKey"
```

### 定义一个配置项

```kconfig
config BOARD_MACRO8
    bool "Macro8"
    depends on SOC_NRF52840_QIAA
```

这类文件不是运行时代码，更像“编译选项声明”。

## 常见修改点

### 修改某个按键做什么

改：

```text
config/macro8.keymap
```

例如把第 5 个键从音量减改成播放/暂停：

```dts
&kp C_PLAY_PAUSE
```

注意保持 `bindings = <...>` 里总数还是 8 个。

### 修改宏输出内容

改 `macros` 里的 `bindings`。

例如把 `macro_npm` 从 `npm run dev` 改为 `npm test`，需要写成类似：

```dts
bindings = <&kp N &kp P &kp M &kp SPACE &kp T &kp E &kp S &kp T &kp RET>;
```

### 修改键盘蓝牙名称

改：

```text
config/macro8.conf
```

```conf
CONFIG_ZMK_KEYBOARD_NAME="NewName"
```

### 修改按键硬件引脚

改：

```text
config/boards/arm/macro8/macro8.dts
```

里的 `input-gpios`。

修改时要保证：

- GPIO 端口和引脚号符合实际 PCB。
- keymap 里的顺序和实体按键顺序一致。
- 按键电路如果不是上拉低有效，标志也要跟着改。

### 修改 RGB 灯数量

改：

```dts
chain-length = <1>;
```

例如 4 颗灯：

```dts
chain-length = <4>;
```

### 修改固件输出格式

当前已经配置为 `.hex`：

```conf
CONFIG_BUILD_OUTPUT_HEX=y
CONFIG_BUILD_OUTPUT_BIN=n
```

GitHub Actions 也配置为：

```yaml
fallback_binary: hex
```

如果以后要回到 `.bin`，需要反向修改这两处。

## 当前项目里最值得注意的点

### 1. 中文注释在 PowerShell 里可能显示乱码

部分文件的中文注释在 PowerShell 输出中显示为乱码，但文件本身可能仍是 UTF-8。建议编辑器统一使用 UTF-8 打开和保存。

### 2. `macro8.conf` 里有些注释和配置可能挤在同一行

如果某行像这样：

```conf
# 注释... CONFIG_SPI=y
```

那么 `CONFIG_SPI=y` 会被当成注释的一部分，不会生效。

当前更重要的是 `macro8_defconfig` 里已经启用了：

```conf
CONFIG_SPI=y
```

所以 RGB 相关构建仍可能正常。但为了可读性，后续可以整理一下注释换行。

### 3. 电池节点定义了，但电池上报被禁用

`macro8.dts` 里有：

```dts
zmk,battery = &vbatt;
```

但 `macro8.conf` 里：

```conf
CONFIG_ZMK_BATTERY_REPORTING=n
```

所以当前不会上报电量。这是有意规避兼容问题。

### 4. `.hex` 输出需要两层配合

Zephyr 层负责生成：

```conf
CONFIG_BUILD_OUTPUT_HEX=y
```

GitHub Actions 层负责打包：

```yaml
fallback_binary: hex
```

只改其中一个，可能会出现“生成了但没打包”或“想打包但没生成”的问题。

## 文件之间的对应关系

```text
build.yaml
  board: macro8
    ↓
config/boards/arm/macro8/macro8.yaml
  id: macro8
    ↓
config/boards/arm/macro8/Kconfig.board
  config BOARD_MACRO8
    ↓
config/boards/arm/macro8/macro8_defconfig
  CONFIG_BOARD_MACRO8=y
  CONFIG_SOC_NRF52840_QIAA=y
    ↓
config/boards/arm/macro8/macro8.dts
  硬件引脚、Flash、RGB、ADC、USB
    ↓
config/macro8.keymap
  8 个按键具体触发什么行为
    ↓
config/macro8.conf
  ZMK 功能开关：USB、BLE、RGB、睡眠等
    ↓
生成 macro8 固件
```

## 如果你把它当成前端项目理解

| 前端概念 | 这个项目里的对应物 |
| --- | --- |
| `package.json` 依赖版本 | `config/west.yml` |
| Vite build 配置 | `.github/workflows/build.yml` 和 `build.yaml` |
| Vue 组件模板 | `.dts` / `.keymap` 的树形节点 |
| `.env` 功能开关 | `.conf` / `defconfig` |
| DOM 节点 | DTS 硬件节点 |
| props | DTS 节点属性 |
| ref | DTS 标签和 `&引用` |
| click handler | keymap binding |
| 第三方库源码 | `.zmk/` |

## 阅读建议

如果你以后想继续改这个键盘，建议按这个顺序看：

1. 先看 `config/macro8.keymap`，因为它最接近“按键功能”。
2. 再看 `config/macro8.conf`，理解启用了哪些 ZMK 功能。
3. 然后看 `config/boards/arm/macro8/macro8.dts`，理解硬件怎么接。
4. 最后看 `macro8_defconfig` 和 Kconfig 文件，理解编译时默认选项。
5. `west.yml`、`module.yml`、`build.yaml`、`build.yml` 主要是构建系统 glue code，平时很少改。

## 小抄

最常见的几个符号：

| 符号 | 意思 |
| --- | --- |
| `#include` | 导入其他定义 |
| `/ { ... };` | DTS 根节点 |
| `label: node {}` | 给节点起一个可引用的标签 |
| `&label` | 引用某个标签 |
| `<1 2 3>` | 整数数组 |
| `compatible` | 用哪个驱动/行为匹配这个节点 |
| `status = "okay"` | 启用硬件节点 |
| `CONFIG_XXX=y` | 启用某功能 |
| `CONFIG_XXX=n` | 禁用某功能 |
| `&kp A` | 按下 A 键 |
| `&out OUT_TOG` | 切换输出目标 |

