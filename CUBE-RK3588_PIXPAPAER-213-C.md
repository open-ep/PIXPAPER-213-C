[![License: GPL v2](https://img.shields.io/badge/License-GPL%20v2-blue.svg)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html)

# PIXPAPER-213-C on CUBE-RK3588 — Linux

| Item | Value |
|---|---|
| Board | [CUBE-RK3588 (device tree `rk3588-b675.dts`)](https://github.com/MayQueenTechCommunity/CUBE-RK3588) |
| Tested | kernel 6.1 |
| Interface | `/dev/spidev4.0` + 3 GPIOs on `gpiochip4` |
| Demo source | [pixpaper-213-c-test-rk3588.c](https://github.com/open-ep/linux-user-space-examples/blob/master/2.13/color/spi/pixpaper-213-c-test-rk3588.c) |

## 1. Wiring

Power the board off before wiring. The panel is **3.3 V only** — never use a 5 V pin.

CUBE-RK3588 has no SPI header. The two UART headers **COM1 / COM2** are turned into SPI4 (step 2),
so the panel goes on those pins. Grid below = the two 2×5 headers as seen in the photo (`·` = not connected):

| Left header | col 1 | col 2 | col 3 | col 4 | col 5 |
|---|---|---|---|---|---|
| top row | **GND** | · | **BUSY** | **CS#** | · |
| bottom row | **3V3** | · | **DC#** | **RST#** | · |

| Right header | col 1 | col 2 | col 3 | col 4 | col 5 |
|---|---|---|---|---|---|
| top row | · | · | **MOSI** | · | · |
| bottom row | · | · | · | **SCK** | · |

| Panel | RK3588 signal |
|---|---|
| MOSI / SCK | SPI4 (`spi4m2` pins) |
| CS# | GPIO4_D4 (driven by `cs-gpios`) |
| DC# | GPIO4_D2 (gpiochip4 line 26) |
| RST# | GPIO4_D3 (gpiochip4 line 27) |
| BUSY | GPIO4_D5 (gpiochip4 line 29) |

<img width="536" alt="photo" src="https://github.com/user-attachments/assets/0b979c3f-dc40-4ad3-9fe2-24afd07a2368" />

## 2. Enable SPI

COM1/COM2 stop working as UARTs after this change.

1. In the kernel source, edit `arch/arm64/boot/dts/rockchip/rk3588-b675.dts` (this is the CUBE-RK3588 board file):

```diff
 &uart5 {
-	status = "okay";
-	pinctrl-0 = <&uart5m0_xfer &uart5m0_ctsn &uart5m0_rtsn>;
+	status = "disabled";
 };

 &uart6 {
-	status = "okay";
-	pinctrl-0 = <&uart6m1_xfer &uart6m1_ctsn &uart6m1_rtsn>;
+	status = "disabled";
 };
```

and append at the end of the file:

```dts
&spi4 {
	status = "okay";
	pinctrl-names = "default";
	pinctrl-0 = <&spi4m2_pins>;
	cs-gpios = <&gpio4 RK_PD4 GPIO_ACTIVE_LOW>;
	num-cs = <1>;

	spi_test@0 {
		compatible = "rockchip,spidev";
		reg = <0>;
		spi-max-frequency = <5000000>;
	};
};
```

2. Rebuild the dtb, copy it over the old one in the boot partition, reboot.

**Check:** `ls /dev/spidev4.0` prints `/dev/spidev4.0`.

## 3. Build and run the demo

> Needs **libgpiod 1.x** — check with `gpioinfo --version` (must say `v1.x`). With libgpiod 2.x the build fails — see Troubleshooting.

> Yocto image: add `IMAGE_INSTALL:append = " libgpiod libgpiod-dev libgpiod-tools packagegroup-core-buildessential"` to `conf/local.conf`, rebuild, then run the commands below on the board (skip the `apt` line).

```bash
sudo apt install -y gcc wget gpiod libgpiod-dev
mkdir -p ~/pixpaper-213c && cd ~/pixpaper-213c
BASE=https://raw.githubusercontent.com/open-ep/linux-user-space-examples/master/2.13/color/spi
wget $BASE/pixpaper-213-c-test-rk3588.c $BASE/png_HEX.h
gcc -o epd_test pixpaper-213-c-test-rk3588.c -lgpiod
sudo ./epd_test
```

## 4. Expected result

The panel flashes for **15–25 s** (normal for a 4-colour full refresh), then shows the sample image (progress bars on a black background). The demo redraws every 5 s until you press **Ctrl+C**.

<img width="640" alt="result" src="https://github.com/user-attachments/assets/224ee0a3-6eda-4a69-8ffe-28d9f717d7a8" />

## 5. Show your own picture

```bash
sudo apt install -y python3-opencv
wget $BASE/png2epd.py
# save your picture as test.png: exactly 250 × 122 px (landscape)
python3 png2epd.py                    # test.png → png_HEX.h
gcc -o epd_test pixpaper-213-c-test-rk3588.c -lgpiod && sudo ./epd_test
```

`png2epd.py` does **not** resize or dither. It keeps only 3 exact colours — black `#000000`, red `#FF0000`, yellow `#FFFF00`; every other colour becomes white. It can also run on your PC; just copy `png_HEX.h` to the board.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `/dev/spidev4.0` missing | SPI is not enabled → redo step 2 and reboot. |
| Compile error `implicit declaration of function 'gpiod_chip_open_by_name'` | Your OS has libgpiod 2.x. Use an image with libgpiod 1.x. |
| `Failed to request GPIO` / line busy | Another driver owns GPIO4_D2/D3/D5. `gpioinfo gpiochip4` shows the user. If you rewire, edit `EPD_DC_PIN` / `EPD_RST_PIN` / `EPD_BUSY_PIN` in the `.c`. |
| Runs, but the screen never changes | Check 3V3 and GND first, then that MOSI and SCK are not swapped. |
| Hangs at 100 % CPU, no flashing | BUSY or RST# wire is loose (BUSY idles high, low = busy). |

## Contributors

Thanks goes to these wonderful people from open source community:

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore-start -->
<!-- markdownlint-disable -->
<table>
  <tbody>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/lc-wang"><img src="https://avatars.githubusercontent.com/u/125327848?v=4" width="100px;" alt="LC Wang"/><br /><sub><b>LC Wang</b></sub></a><br /><a href="https://github.com/wigcheng/open-epd/commits?author=lc-wang" title="Code">💻</a></td>
    </tr>
  </tbody>
</table>

<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

<!-- ALL-CONTRIBUTORS-LIST:END -->

---
