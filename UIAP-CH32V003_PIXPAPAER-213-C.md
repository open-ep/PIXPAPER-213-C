[![License: GPL v2](https://img.shields.io/badge/License-GPL%20v2-blue.svg)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html)

# PIXPAPER-213-C on UIAPduino Pro Micro CH32V003 — Arduino

| Item | Value |
|---|---|
| Board | [UIAPduino Pro Micro CH32V003 V1.4](https://www.uiap.jp/en/uiapduino/pro-micro/ch32v003/v1dot4) — RISC-V 48 MHz, 16 KB flash / 2 KB SRAM |
| Toolchain | Arduino IDE (Windows / Linux), flashed over USB-C, no programmer |
| Source | [arduino-user-space-examples → uiap/pixpaper-213-c](https://github.com/open-ep/arduino-user-space-examples/tree/main/uiap/pixpaper-213-c) |

2 KB SRAM is too small for a framebuffer, so the picture is converted on your PC into a packed array in flash and streamed straight into the panel.

## 0. Switch the board to 3.3 V (one-time, required)

The board ships at **5 V logic**, which over-drives the 3.3 V panel (faint image, and 5 V back-feeds the panel's 3.3 V rail).

1. Find the **Volt-Sel** solder jumper (see the [board page](https://www.uiap.jp/en/uiapduino/pro-micro/ch32v003/v1dot4)).
2. Cut the **5V** side with a knife — a minimal cut, there are traces right above and below it.
3. Bridge the **3.3V** side with solder.

**Check:** with USB plugged in, a multimeter between the **3V3** pin and **GND** reads **3.3 V**, and the 5V side of the jumper is open (no beep in continuity mode).

## 1. Wiring

| Panel | UIAPduino pin (silkscreen) | CH32V003 GPIO |
|---|---|---|
| 3V3 | 3V3 | – |
| GND | GND | – |
| MOSI | 8 | PC6 |
| SCK | 7 | PC5 |
| CS# | 6 | PC4 |
| DC# | 5 | PC3 |
| RST# | 10 | PD0 |
| BUSY | 12 (A3) | PD2 |

## 2. Arduino IDE setup (once)

1. Install the Arduino IDE.
2. **File → Preferences → Additional boards manager URLs**, add:
   `https://github.com/YuukiUmeta-UIAP/board_manager_files/raw/main/package_uiap.jp_index.json`
3. **Tools → Board → Boards Manager**, search `UIAPduino`, install.
4. **Tools → Board** → `Pro Micro CH32V003`, Board Version `V1.4`.
5. Keep **Tools → Optimize** at `Smallest (-Os) default`. **Do not pick an LTO option** — LTO firmware does not boot on this board.

## 3. How to flash (every time)

1. Hold the **reset** button, plug in USB, release it right away. The board appears as a HID device `32V003`.
2. Wait until the OS has finished setting it up, then press **Upload**.
3. Done when the IDE prints `Image written.`

The sketches have no USB function, so the board drops off USB once it runs. Repeat step 1 to re-flash.

## 4. First test: quick-update (picture + colour bars)

```bash
mkdir pixpaper_213_color && cd pixpaper_213_color
BASE=https://raw.githubusercontent.com/open-ep/arduino-user-space-examples/refs/heads/main/uiap/pixpaper-213-c
wget $BASE/quick-update/pixpaper_213_color.ino $BASE/quick-update/img_packed.h
```

Open `pixpaper_213_color.ino` in the Arduino IDE (the folder name must match the `.ino` name), flash it (step 3).

**Expected:** the panel flashes for **15–25 s** (normal for 4 colours), shows the bundled picture, then every 30 s alternates with black / white / red / yellow bars.

## 5. Show any picture: mix-color (with dithering)

```bash
mkdir pixpaper_213_mix_color && cd pixpaper_213_mix_color
wget $BASE/mix-color/pixpaper_213_mix_color.ino $BASE/mix-color/img_packed.h $BASE/mix-color/png2packed_dither.py
sudo apt install -y python3-opencv
```

1. Crop your picture to about **2:1 landscape** (the converter resizes to exactly 250×122; other shapes get squashed).
2. Convert, and **check `preview.png` on the PC first** — it shows exactly what the panel will show:
   ```bash
   python3 png2packed_dither.py your_image.png --boost --preview preview.png -o img_packed.h
   ```
3. Flash `pixpaper_213_mix_color.ino` (step 3). The new `img_packed.h` sits next to it.

Pick the option by picture type:

| Picture | Options | What it does |
|---|---|---|
| Logo, UI, flat-colour art | `--no-dither` | nearest of the 4 colours; flat areas stay clean |
| Photo, illustration | `--boost` | blue → gray, saturation ×1.8, contrast ×1.15, then dither |
| Anime / thin outlines | `--boost --lines` | as above + re-draws edges in black |
| Gradient, plain dither | *(no option)* | Floyd-Steinberg only |
| Not sure | make `--no-dither` and `--boost`, compare the previews | |

The panel has only black / white / yellow / red. Dithering fakes orange (red+yellow), pink (red+white), gray (black+white), brown (red+black). There is no blue ink.

**Expected:** after the 15–25 s refresh the panel shows your picture (the bundled `img_packed.h` is an orange / pink / brown / gray test chart), then alternates with colour bars every 30 s like quick-update.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Unknown USB device / `error -71` in `dmesg` | Plug through a **USB 2.0 hub**; some USB 3.0 ports cannot talk to the software-USB bootloader. |
| Upload fails | Redo the reset-plug ritual (step 3); the running sketch is not a USB device. |
| Board does not run after upload | You built with LTO → set Optimize back to `Smallest (-Os) default`. |
| Image very faint | Board is still at 5 V → step 0. |
| Screen never changes | Check 3V3 / GND, then MOSI (8) and SCK (7) are not swapped. |

## Zephyr?

Not for this panel on this board: one 4-colour frame (7750 B) does not fit in 16 KB flash next to Zephyr. The mono [PIXPAPER-213-M](https://github.com/open-ep/PIXPAPER-213-M/blob/main/UIAP-CH32V003_PIXPAPAER-213-M.md) runs on Zephyr here. For the 4-colour panel on Zephyr, use the [W6300-EVB-Pico2 guide](ZEPHYR-W6300-EVB-PICO2_PIXPAPAER-213-C.md).

## Contributors

Thanks goes to these wonderful people from open source community:

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore-start -->
<!-- markdownlint-disable -->
<table>
  <tbody>
    <tr>
        <td align="center" valign="top" width="14.28%"><a href="https://github.com/wigcheng"><img src="https://avatars.githubusercontent.com/u/7148592?v=4" width="100px;" alt="Wig Cheng"/><br /><sub><b>Wig Cheng</b></sub></a><br /><a href="https://github.com/wigcheng/open-ep/commits?author=wigcheng" title="Code">💻</a></td>
    </tr>
  </tbody>
</table>

<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

<!-- ALL-CONTRIBUTORS-LIST:END -->

---
