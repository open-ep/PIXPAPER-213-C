[![License: GPL v2](https://img.shields.io/badge/License-GPL%20v2-blue.svg)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html)

## Overview

|Platform|Tested|
|---|---|
| UIAPduino Pro Micro CH32V003 V1.4 | &#10004;|

The [UIAPduino Pro Micro CH32V003](https://www.uiap.jp/en/uiapduino/pro-micro/ch32v003/v1dot4) is a tiny RISC-V board (48 MHz, 16KB flash / 2KB SRAM) in the Pro Micro form factor, programmable from the Arduino IDE over a single USB Type-C cable — no external programmer needed.

Note that CH32V003 has only 2KB SRAM, so there is no framebuffer: the image is converted on the host PC into a pre-packed 2bpp array stored in flash, and streamed directly into the panel controller RAM over bit-banged SPI.

All example sources live in
[open-ep/arduino-user-space-examples → uiap/pixpaper-213-c](https://github.com/open-ep/arduino-user-space-examples/tree/main/uiap/pixpaper-213-c).

## Hardware Preparison

**IMPORTANT: the board must be reworked to 3.3V before connecting the panel.** The UIAPduino ships at 5V by default (GPIO logic = 5V), which over-drives the 3.3V PIXPAPER panel — the image comes out faint and the 5V logic back-feeds the panel's 3.3V rail. Cut the 5V side of the Volt-Sel solder jumper with a knife (minimal cut — there are traces right above and below it), bridge the 3.3V side with solder, then verify with a multimeter that the 5V side is really open. See the [official board page](https://www.uiap.jp/en/uiapduino/pro-micro/ch32v003/v1dot4) for the jumper location.

Firstly, connecting the PIXPAPER-213-C's connector to the programming cable we've provided. Connect the other end of the cable to the corresponding pins, matching the colors as defined.

<img width="640" alt="image" src="https://github.com/user-attachments/assets/278a84f1-97a0-4ab5-ac1d-c94a1133bda3" />


Then, connect to the UIAPduino specific PINs as follows:

|PIXPAPER-213-C Pinout|UIAPduino Pin assignment (silkscreen)|CH32V003 GPIO|
|---|---|---|
| 3V3 | 3V3 | - |
| GND | GND | - |
| MOSI | 8 | PC6 |
| SCK | 7 | PC5 |
| CS | 6 | PC4 |
| DC | 5 | PC3 |
| RST | 10 | PD0 |
| BUSY | 12 (A3) | PD2 |

## Arduino Environment Setup

        1. Install Arduino IDE (Windows/Ubuntu both work)
        2. File -> Preferences -> Additional boards manager URLs, add:
           https://github.com/YuukiUmeta-UIAP/board_manager_files/raw/main/package_uiap.jp_index.json
        3. Tools -> Board -> Boards Manager, search "UIAPduino" and install it
        4. Tools -> Board -> select "Pro Micro CH32V003", Board Version "V1.4"

        Note: keep Tools -> Optimize at the default "Smallest (-Os) default".
        Do NOT select the LTO options -- LTO-built firmware does not boot on this board.

## Flashing (common to all examples)

        While pressing the reset button, connect the board to USB, then immediately
        release the reset button. The board enumerates as an HID device ("32V003").
        After the OS has finished setting up the device, press Upload in the Arduino IDE.
        When the message "Image written." is displayed, writing is complete.

        Note that the sketches have no USB function, so the board disappears from USB
        once it runs -- repeat the reset ritual every time you want to re-flash.
        If the device fails to enumerate (unknown USB device / error -71 in dmesg),
        connect it through a USB 2.0 hub instead of a USB 3.0 port: the bootloader's
        software USB is timing-sensitive and some xHCI hosts cannot enumerate it directly.

---

## Arduino Example 1: mix-color — show any picture (recommended)

The panel physically has only 4 pixel states (black / white / yellow / red) and
cannot mix colors per pixel. This example ships an image converter with
Floyd-Steinberg dithering that simulates in-between colors spatially:
red+yellow reads as orange, red+white as pink, black+white as gray, red+black
as brown. Blue does not exist in the ink — blue-ish content is rendered as
equal-luminance gray.

The bundled sample image is a gradient test chart (orange / pink / brown /
gray ramp) that demonstrates every mixed color at once. A full 4-color refresh
takes about 15-25 seconds — this is normal for this panel, be patient after
power-on.

### Step 1. Download and flash the sketch

The .ino must live in a folder with the same name, as the Arduino IDE requires:

        $ mkdir pixpaper_213_mix_color && cd pixpaper_213_mix_color
        $ wget https://raw.githubusercontent.com/open-ep/arduino-user-space-examples/refs/heads/main/uiap/pixpaper-213-c/mix-color/pixpaper_213_mix_color.ino
        $ wget https://raw.githubusercontent.com/open-ep/arduino-user-space-examples/refs/heads/main/uiap/pixpaper-213-c/mix-color/img_packed.h

        Then open pixpaper_213_mix_color.ino in the Arduino IDE and upload.

### Step 2. Convert your own image

Get the converter (python3 + opencv required):

        $ sudo apt install python3-opencv
        $ wget https://raw.githubusercontent.com/open-ep/arduino-user-space-examples/refs/heads/main/uiap/pixpaper-213-c/mix-color/png2packed_dither.py

Before converting, crop your source image close to the panel's 250:122 (about
2:1) aspect ratio — the converter resizes to exactly 250x122, so a square or
portrait source would come out squashed. Any image editor works; crop so the
subject fills the frame (250x122 has very little room for background detail).

**Always generate a preview and check it on the PC before flashing anything**
(`--preview` renders exactly what the panel will show, at 2x):

**Recipe A — no dithering (flat-color artwork, logos, UI screens):**

        $ python3 png2packed_dither.py your_image.png --no-dither --preview preview.png -o img_packed.h

Every pixel is quantized to the nearest of the 4 panel colors, no error
diffusion. Use this when the artwork is already drawn in (or close to) the 4
panel colors: large flat areas stay perfectly clean, while dithering would
speckle them with noise dots.

**Recipe B — plain dithering (photos, gradients):**

        $ python3 png2packed_dither.py your_image.png --preview preview.png -o img_packed.h

Floyd-Steinberg error diffusion (serpentine scan). Smooth gradients become
fine 2-color checkerboards that the eye blends back into intermediate tones.
This is the default mode when no option is given.

**Recipe C — enhanced dithering (most photos and illustrations look best here):**

        $ python3 png2packed_dither.py your_image.png --boost --preview preview.png -o img_packed.h

`--boost` pre-processes the image before dithering, three things at once:

        1. Blue/cyan hues are neutralized to equal-luminance gray. The panel has
           no blue ink: without this step, blue areas either vanish into white
           or turn into dark-red noise. As gray, they dither into a black/white
           pattern that keeps the shape clearly visible.
        2. Saturation x1.8 -- washed-out colors don't survive 4-color
           quantization; boosting first keeps yellows and reds vivid.
        3. Contrast x1.15 -- keeps outlines readable after error diffusion.

**Recipe D — line art rescue (anime / character art with thin outlines):**

        $ python3 png2packed_dither.py your_image.png --boost --lines --preview preview.png -o img_packed.h

`--lines` runs edge detection (Canny) and overlays the detected edges in solid
black on top of the dithered result. Thin dark outlines tend to dissolve into
dither noise; this re-inks them. Avoid it for photos — photographic edges turn
into harsh black scratches.

### Which recipe for which image?

| Source image | Recipe |
| --- | --- |
| Logo / UI / flat-color poster | A (`--no-dither`) |
| Photo | C (`--boost`), crop tight on the subject |
| Illustration with bold shapes | C (`--boost`) |
| Anime / character art, thin outlines | D (`--boost --lines`) |
| Not sure | Generate A and C, compare the two previews |

After converting, replace `img_packed.h` next to the sketch, rebuild and
re-flash (the reset ritual again). The firmware is identical for every recipe
— to the sketch, a dithered image is just different pixel data.

Expection results: <br>

(photo / video placeholder)

---

## Arduino Example 2: quick-update — bundled picture + color-bar cycle

Displays a bundled picture, alternating with a black/white/red/yellow color-bar
test pattern every 30 seconds. Useful as the first smoke test of the wiring
and the 3.3V rework, before converting your own images.

Step 1. Download the sketch:

        $ mkdir pixpaper_213_color && cd pixpaper_213_color
        $ wget https://raw.githubusercontent.com/open-ep/arduino-user-space-examples/refs/heads/main/uiap/pixpaper-213-c/quick-update/pixpaper_213_color.ino
        $ wget https://raw.githubusercontent.com/open-ep/arduino-user-space-examples/refs/heads/main/uiap/pixpaper-213-c/quick-update/img_packed.h

        Then open pixpaper_213_color.ino in the Arduino IDE and upload.

Step 2. (optional) This example has its own minimal converter
        (nearest-color only, equivalent to mix-color's Recipe A):

        $ wget https://raw.githubusercontent.com/open-ep/arduino-user-space-examples/refs/heads/main/uiap/pixpaper-213-c/quick-update/png2packed_color.py
        $ python3 png2packed_color.py your_image.png -o img_packed.h

        For dithering and the enhancement options, use the mix-color converter
        from Example 1 -- the generated img_packed.h is interchangeable
        between both sketches.

Expection results: <br>

(photo / video placeholder)

---

## Zephyr RTOS

There is no Zephyr example for the 4-color panel on this board: a 2bpp frame is
7750 bytes and does not fit in 16KB flash next to the Zephyr kernel. The
monochrome PIXPAPER-213-M panel is supported on Zephyr (boot image + interactive
UART console) — see
[open-ep/zephyr-user-space-examples → boards/uiapduino](https://github.com/open-ep/zephyr-user-space-examples/tree/main/boards/uiapduino)
and the [PIXPAPER-213-M UIAP guide](https://github.com/open-ep/PIXPAPER-213-M/blob/main/UIAP-CH32V003_PIXPAPAER-213-M.md).

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
