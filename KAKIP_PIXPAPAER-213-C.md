[![License: GPL v2](https://img.shields.io/badge/License-GPL%20v2-blue.svg)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html)

# PIXPAPER-213-C on Kakip (RZ/V2H) — Linux

| Item | Value |
|---|---|
| Board | [Kakip AI SBC (Renesas RZ/V2H)](https://www.kakip.ai/) |
| Tested | kernel 5.10 (6.1: not tested yet) |
| Interface | `/dev/spidev0.0` + 3 GPIOs on `gpiochip0` |
| Demo source | [pixpaper-213-c-test-kakip.c](https://github.com/open-ep/linux-user-space-examples/blob/master/2.13/color/spi/pixpaper-213-c-test-kakip.c) |

## 1. Wiring

Power the board off before wiring. The panel is **3.3 V only** — never use a 5 V pin.

<img src="images/wiring/kakip.svg" width="640" alt="Kakip (RZ/V2H) wiring">

| Panel | CN6 pin | SoC signal |
|---|---|---|
| 3V3 | 1 | 3.3 V |
| GND | 9 (any GND) | GND |
| MOSI | 19 | P9_0 / RSPI0 MOSI |
| SCK | 23 | P9_2 / RSPI0 RSPCK |
| CS# | 24 | P9_3 / RSPI0 SSL0 |
| DC# | 29 | P7_4 (gpio line 60) |
| RST# | 31 | P10_7 (gpio line 87) |
| BUSY | 37 | P11_3 (gpio line 91) |

<details><summary>Photo of a finished setup</summary>

<img width="640" alt="photo" src="https://github.com/user-attachments/assets/051eab7c-4836-4256-8eaf-bb09fbef7d93" />
</details>

## 2. Enable SPI

The stock device tree enables `&spi0` (RSPI0, pins P9_0–P9_3) but has no device on it.
Add a `spidev` child node, rebuild the dtb, and boot with it.

1. In the kernel source, open `arch/arm64/boot/dts/renesas/kakip-es1.dts`, find the existing `&spi0 { ... };` node, and add the `spi@0` block inside it:

```dts
&spi0 {
	/* ...keep the existing lines... */
	status = "okay";

	spi_dev0: spi@0 {			/* add this block */
		compatible = "rohm,dh2228fv";	/* any spidev-bound compatible */
		reg = <0>;
		spi-max-frequency = <10000000>;
	};
};
```

2. Rebuild the dtb (`make dtbs`), copy `kakip-es1.dtb` over the one in `/boot` on the rootfs, reboot.

**Check:** `ls /dev/spidev0.0` prints `/dev/spidev0.0`.

## 3. Build and run the demo

> Needs **libgpiod 1.x** — check with `gpioinfo --version` (must say `v1.x`). With libgpiod 2.x the build fails — see Troubleshooting.

```bash
sudo apt install -y gcc wget gpiod libgpiod-dev
mkdir -p ~/pixpaper-213c && cd ~/pixpaper-213c
BASE=https://raw.githubusercontent.com/open-ep/linux-user-space-examples/master/2.13/color/spi
wget $BASE/pixpaper-213-c-test-kakip.c $BASE/png_HEX.h
gcc -o epd_test pixpaper-213-c-test-kakip.c -lgpiod
sudo ./epd_test
```

## 4. Expected result

The panel flashes for **15–25 s** (normal for a 4-colour full refresh), then shows the sample image. The demo redraws every 30 s until you press **Ctrl+C**.

https://github.com/user-attachments/assets/9bc75d90-b77c-44e8-84ba-26e8f9112780

## 5. Show your own picture

```bash
sudo apt install -y python3-opencv
wget $BASE/png2epd.py
# save your picture as test.png: exactly 250 × 122 px (landscape)
python3 png2epd.py                    # test.png → png_HEX.h
gcc -o epd_test pixpaper-213-c-test-kakip.c -lgpiod && sudo ./epd_test
```

`png2epd.py` does **not** resize or dither. It keeps only 3 exact colours — black `#000000`, red `#FF0000`, yellow `#FFFF00`; every other colour becomes white. It can also run on your PC; just copy `png_HEX.h` to the board.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `/dev/spidev0.0` missing | SPI is not enabled → redo step 2 and reboot. |
| Compile error `implicit declaration of function 'gpiod_chip_open_by_name'` | Your OS has libgpiod 2.x. Use an image with libgpiod 1.x. |
| `Failed to request GPIO` / line busy | Another driver owns P7_4/P10_7/P11_3. `gpioinfo gpiochip0` shows the user. If you rewire, edit `EPD_DC_PIN` / `EPD_RST_PIN` / `EPD_BUSY_PIN` in the `.c`. |
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
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/lc-wang"><img src="https://avatars.githubusercontent.com/u/125327848?v=4" width="100px;" alt="LC Wang"/><br /><sub><b>LC Wang</b></sub></a><br /><a href="https://github.com/wigcheng/open-ep/commits?author=lc-wang" title="Code">💻</a></td>
    </tr>
  </tbody>
</table>

<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

<!-- ALL-CONTRIBUTORS-LIST:END -->

---
