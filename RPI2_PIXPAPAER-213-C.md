[![License: GPL v2](https://img.shields.io/badge/License-GPL%20v2-blue.svg)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html)

## Overview

## Hardware Preparison

[Raspberry PI 2 Model](https://www.raspberrypi.com/products/raspberry-pi-2-model-b/) is a very low-power ARM32 Cortex-A7 platform released in 2014. Even after 10 years, it remains a top choice for simple IoT applications and continues to be available on the official website. Its operating system is also actively maintained. If you’re looking for an energy-efficient and affordable solution, especially when paired with the PIXPAPER-213-C for simple applications, the Raspberry PI 2 is an excellent option.

To start, use the raspi-config command to open the configuration interface and enable SPI.
Note that our Raspberry Pi OS is based on the November 2024 release.


Firstly, connecting the PIXPAPER-213-C's connector to the programming cable we've provided. Connect the other end of the cable to the corresponding pins, matching the colors as defined.

<img width="640" alt="image" src="https://github.com/user-attachments/assets/278a84f1-97a0-4ab5-ac1d-c94a1133bda3" />

Then, connect to the Raspberry PI specific PINs of 40-PIN header as follows:
<img width="640" alt="image" src="https://github.com/user-attachments/assets/e5918ec0-b46e-4a1f-95e9-53da998e52f0" />



## Driver Installation instructions

Raspberry PI OS can dynamic enable/disable SPI interface, no need tweak kernel source code.
Issue "rasp-config" command into the control GUI and following steps as below:

Step 1. Selecting **Interface Options** <br>
<img width="600" alt="image" src="https://github.com/user-attachments/assets/d129666a-a945-4662-bb39-67629a0bf9f4" />

Step 2. Selecting **SPI** to enable SPI interface <br>
<img width="600" alt="image" src="https://github.com/user-attachments/assets/8dfb5ce3-cb5b-4ae9-b4e2-10a0cda5a5bd" />

Step 3. Selecting **YES** to make sure your change <br>
<img width="600" alt="image" src="https://github.com/user-attachments/assets/ab50b6ee-20a9-407c-940a-522bf97c3a4a" />

Step 4. It will showing the SPI enabled message, just select OK to close the  <br>
<img width="600" alt="image" src="https://github.com/user-attachments/assets/07d52120-d5ab-4548-bf98-a2ddc9263794" />

Step 5. Reboot the device and check spi device node is existed or not.

    ls /dev/spidev0.0
 

## User-Space Utility instructions (Linux OS)



Step 1. Install necessary packages

        Ubuntu/Debian:
        $ sudo apt install gpiod libgpiod-dev


Step 2. Prepare a 250x122 size picture what you want to showing, then make a image raw data based header file

        Download the PNG to RAW converter base on python3, remember to install opencv package first
        $ sudo apt install python3-opencv
        $ wget https://raw.githubusercontent.com/open-EPD/user-space-examples/refs/heads/master/2.13/color/spi/png2epd.py

        Then, rename your PNG file as test.png, and excute the python script
        $ python3 png2epd.py

        It will generate a output file: png_HEX.h, the copy the same folder witth pixpaper-213-c-test-rpi2.c.
        Note that this step can running on host PC side or target board, but png_HEX.h must be put into the folder with c file together before 
        compiling.

        Download a sample header file:
        wget https://raw.githubusercontent.com/open-EPD/user-space-examples/refs/heads/master/2.13/color/spi/png_HEX.h


Step 3. Please download the utility source code in the rootfs of Raspberry PI, then compile it and execute the compiled executable file.

        # wget https://raw.githubusercontent.com/open-EPD/user-space-examples/refs/heads/master/2.13/color/spi/pixpaper-213-c-test-rpi2.c
        # gcc -o epd_test pixpaper-213-c-test-rpi2.c -lgpiod
        # ./epd_test

        Note that if your wired connection is different with chapter 1 "Hardware Preparison", especially DC# PIN, RST# PIN, and BUSY PIN, also can issue command 'gpioinfo' to check the gpip pin detail. 
        Please modify the specific macros definition of pixpaper-213-c-test-rpi2.c:

        #define EPD_GPIO_CHIP "gpiochip0"
        #define EPD_DC_PIN 23
        #define EPD_RST_PIN 24
        #define EPD_BUSY_PIN 25


Expection results: <br>

https://github.com/user-attachments/assets/9a45a8c4-d8d0-4d60-a007-33870f8bfbd4

## Contributors

Thanks goes to these wonderful people from open source community:

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore-start -->
<!-- markdownlint-disable -->
<table>
  <tbody>
    <tr>
        <td align="center" valign="top" width="14.28%"><a href="https://github.com/wigcheng"><img src="https://avatars.githubusercontent.com/u/7148592?v=4" width="100px;" alt="Wig Cheng"/><br /><sub><b>Wig Cheng</b></sub></a><br /><a href="https://github.com/wigcheng/open-epd/commits?author=wigcheng" title="Code">💻</a></td>
    </tr>
  </tbody>
</table>

<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

<!-- ALL-CONTRIBUTORS-LIST:END -->

---

