# Flugprojekt UWB Setup

Im folgenden sind die notwendigen Schritte beschrieben, um die DWM1001-Dev-Kits in Betrieb zu nehmen.
Die ursprüngliche Dokumentation ist am Ende zu finden.

Die folgenden Schritte sind Dokumentation für eine native Linuxumgebung oder WSL 2 unter Windows.
Im WSL sollte eine Linuxdistribution installiert sein.

**WICHTIG für WSL-Nutzer**
Alle Schritte in der WSL müssen in einem Linuxverzeichnis sein, spricht nicht innerhalb vom ```mnt``` Verzeichnis!

## Schnellstart und Einrichtung der Flash-Tools

1. Herunterladen und Installation von Apache Newt (Für andere Installationsoptionen siehe: https://mynewt.apache.org/latest/newt/install/newt_linux.html)

```
wget https://raw.githubusercontent.com/JuulLabs-OSS/binary-releases/master/mynewt-newt-tools_1.10.0/newt_1.10.0-1_amd64.deb

dpkg -i newt_1.10.0-1_amd64.deb
```

2. Herunterladen und Installation von nrfjfrog (Mehr Infos dazu hier: [Nordic Cmd Tools](https://www.nordicsemi.com/Software-and-tools/Development-Tools/nRF-Command-Line-Tools/Download))

```
# this is for amd64
wget https://nsscprodmedia.blob.core.windows.net/prod/software-and-other-downloads/desktop-software/nrf-command-line-tools/sw/versions-10-x-x/10-24-2/nrf-command-line-tools_10.24.2_amd64.deb

dpkg -i nrf-command-line-tools_10.24.2_amd64.deb

# install recommended JLink 
apt install /opt/nrf-command-line-tools/share/JLink_Linux_V794e_arm64.deb --fix-broken
```

3. Klonen des Repositorys

Hinweis: Hier Klonen mittels SSH, hierfür muss ein Public Key bei Github hinterlegt sein.

```no-highlight
git clone git@github.com:j-peller/uwb-apps.git
cd uwb-apps
```

4. Herunterladen von Abhängigkeiten

```no-highlight
newt upgrade
```

Nach dem Setup restliche Abhängigkeiten herunterladen:

```
    repos/decawave-uwb-core/setup.sh
    newt upgrade
```

Patches ausführen:

```
cd repos/apache-mynewt-core/
git apply ../decawave-uwb-core/patches/apache-mynewt-core/mynewt_1_7_0_*
git apply ../decawave-uwb-core/patches/apache-mynewt-core/mynewt_1_8_*
cd -
```

5. Defaultimage löschen

Nun muss unser Dev-Kit mit der USB-Schnittstelle an unseren Hostrechner angeschlossen werden.

Falls man dies unter WSL macht muss man noch folgendes machen:

Installation von usbipd nach der Anleitung hier: https://github.com/dorssel/usbipd-win?tab=readme-ov-file  
Dann muss man mit besagter Anleitung das angeschlossene Dev-Kit in der WSL sichtbar machen mit den bind und attach Kommandos von usbipd.

Nun wenn alles angeschlossen und eingerichtet ist, kann man die Defaultinstallation so löschen:

```
nrfjprog -f NRF52 -e
```

**Warnung für WSL-Nutzer**
Unter WSL und Nutzung von usbipd kann sich das USB-Device disconnecten. Wenn das passiert, einfach neu den bind und attach Befehl von usbipd ausführen.

6. Installation der ARM-Toolchain

Hier muss die korrekte Version heruntergeladen werden. Download für Linux 32-bit: https://developer.arm.com/downloads/-/gnu-rm/5-2016-q3-update

WSL-Nutzer müssen diese heruntergeladene Datei in ein Linuxverzeichnis verschieben.

Falls das Hostsystem 64bit ist, muss folgendes ausgeführt werden (aller Wahrscheinlichkeit nach trifft dieser Schritt zu!):

```
dpkg --add-architecture i386

echo "deb http://security.ubuntu.com/ubuntu focal-security main universe" | sudo tee /etc/apt/sources.list.d/focal-security.list

apt update
apt install libc6:i386 libstdc++6:i386 libncurses5:i386
```

Nach dem Herunterladen der Toolchain und möglicherweise der Abhängigkeiten für 64bit Systeme kann man wie folgt die Installation vollenden:

```
bzip2 -d gcc-arm-none-eabi-5_4-2016q3-20160926-linux.tar.bz2
tar -xvf gcc-arm-none-eabi-5_4-2016q3-20160926-linux.tar

mv gcc-arm-none-eabi-5_4-2016q3/ /opt/

export PATH=/opt/gcc-arm-none-eabi-5_4-2016q3/bin:$PATH
source ~/.bashrc

/* check if working */
arm-none-eabi-gcc --version
```

## Bauen der zu flashenden Artefakte

Es muss immer der Bootloader gebaut und geflasht werden. Dann kann man den jeweiligen Schritt machen, den man braucht, sprich Master, Slave oder Tag.

### Build bootloader

Builds bootloader applicaiton for the DWM1001 target. Bootloader should be flashed first for each DWM1001 module regardless of the role:

```no-highlight
newt target create dwm1001_boot
newt target set dwm1001_boot app=@mcuboot/boot/mynewt
newt target set dwm1001_boot bsp=@decawave-uwb-core/hw/bsp/dwm1001
newt target set dwm1001_boot build_profile=optimized
newt build dwm1001_boot

# Execute for _each_ DWM1001 module
newt load dwm1001_boot
```

### Master module

Only one module among anchors which acts as a master is allowed per network. In this configuration it is possible to have 4 anchors minimum and up to 8 modules:

```no-highlight
# dont forget to flash the boot loader first
newt load dwm1001_boot

# now start flashing twr firmware 
newt target create nrng_master
newt target set nrng_master app=apps/twr_nranges_tdma
newt target set nrng_master bsp=@decawave-uwb-core/hw/bsp/dwm1001
newt target set nrng_master build_profile=optimized

# Disable additional debug output
newt target amend nrng_master syscfg=NRNG_VERBOSE=0:SURVEY_ENABLED=0

# Disable RTT enable UART
newt target amend nrng_master syscfg=CONSOLE_RTT=0:CONSOLE_UART=1

# Configure number of nodes and frames
newt target amend nrng_master syscfg=NRNG_NFRAMES=16:NRNG_NNODES=8:NRNG_NTAGS=4:NODE_START_SLOT_ID=0:NODE_END_SLOT_ID=7

# Configure role
newt target amend nrng_master syscfg=PANMASTER_ISSUER=1:UWB_TRANSPORT_ROLE=0

newt create-image nrng_master 1.0.0

# this flashes the image onto the dwm1001, ensure usb connectivity
newt load nrng_master
```

### Slave modules

All other non-master anchors should be configured as slaves:

```no-highlight
# dont forget to flash the boot loader first
newt load dwm1001_boot

# now start flashing twr firmware 
newt target create nrng_slave
newt target set nrng_slave app=apps/twr_nranges_tdma
newt target set nrng_slave bsp=@decawave-uwb-core/hw/bsp/dwm1001
newt target set nrng_slave build_profile=optimized

# Disable additional debug output
newt target amend nrng_slave syscfg=NRNG_VERBOSE=0:SURVEY_ENABLED=0

# Disable RTT and UART
newt target amend nrng_slave syscfg=CONSOLE_RTT=0:CONSOLE_UART=1

# Configure number of nodes and frames
newt target amend nrng_slave syscfg=NRNG_NFRAMES=16:NRNG_NNODES=8:NRNG_NTAGS=4:NODE_START_SLOT_ID=0:NODE_END_SLOT_ID=7

# Configure role
newt target amend nrng_slave syscfg=NRANGES_ANCHOR=1:UWB_TRANSPORT_ROLE=0

newt create-image nrng_slave 1.0.0

# this flashes the image onto the dwm1001, ensure usb connectivity
newt load nrng_slave
```

### Tag module

Only one module acts as a tag and is mounted on the drone, connected to the Raspberry Pi 5 via UART

```no-highlight
# dont forget to flash the boot loader first
newt load dwm1001_boot

# now start flashing twr firmware 
newt target create nrng_tag
newt target set nrng_tag app=apps/twr_nranges_tdma
newt target set nrng_tag bsp=@decawave-uwb-core/hw/bsp/dwm1001
newt target set nrng_tag build_profile=optimized

# Disable additional debug output
newt target amend nrng_tag syscfg=NRNG_VERBOSE=0:SURVEY_ENABLED=0

# Disable RTT and UART
newt target amend nrng_tag syscfg=CONSOLE_RTT=0:CONSOLE_UART=1

# Configure number of nodes and frames
newt target amend nrng_tag syscfg=NRNG_NFRAMES=16:NRNG_NNODES=8:NRNG_NTAGS=4:NODE_START_SLOT_ID=0:NODE_END_SLOT_ID=7

# Configure role
newt target amend nrng_tag syscfg=UWB_TRANSPORT_ROLE=1

newt create-image nrng_tag 1.0.0

# this flashes the image onto the dwm1001, ensure usb connectivity
newt load nrng_tag
```

**!!!!! Don't forget to load bootloader for each module !!!!!**

## Testen der Distanzmessung

1. Ein Dev-Kit als Tag flashen und einen zweiten als Master flashen.

2. Tag via USBmit dem Hostrechner verbinden und Master mit einer Stromquelle verbinden. 

2. Verbinden des Tags an den Hostrechner mit der seriellen Schnittstelle via UART (typically /dev/ttyACMX or /dev/ttyUSBX):

Für Putty serielle Schnittstelle auswählen und als Speed 460800 einstellen.

Hinweis für Windowsnutzer: Im Gerätemanager kann man den COM-Port herausfinden, an dem der Tag hängt (COM1, COM2 oder COM3 usw.).

Minicom Kommando (/dev/ttyACM0 muss ggf. mit der korrekten Schnittstelle ersetzt werden):
```
minicom -D /dev/ttyACM0 -b 460800
```

In der interaktiven Konsole sollten, soweit Master und Tag funktionsfähig sind, die Daten der Distanzmessung zu sehen sein.

---

Ab hier die ursprüngliche Dokumentation:

<!--
#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
#  KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
#
-->

# Decawave UWB Applications

## Overview

This distribution contains the example applications `twr_nranges_tdma` for the DWM1001.

## Getting started

1. Download and install Apache Newt or follow the guide here: https://mynewt.apache.org/latest/newt/install/newt_linux.html

```
wget https://raw.githubusercontent.com/JuulLabs-OSS/binary-releases/master/mynewt-newt-tools_1.10.0/newt_1.10.0-1_amd64.deb

dpkg -i newt_1.10.0-1_amd64.deb
```

2. Download and install nrfjprog ([Nordic Cmd Tools](https://www.nordicsemi.com/Software-and-tools/Development-Tools/nRF-Command-Line-Tools/Download))

```
# this is for amd64
wget https://nsscprodmedia.blob.core.windows.net/prod/software-and-other-downloads/desktop-software/nrf-command-line-tools/sw/versions-10-x-x/10-24-2/nrf-command-line-tools_10.24.2_amd64.deb

dpkg -i nrf-command-line-tools_10.24.2_amd64.deb

# install recommended JLink 
apt install /opt/nrf-command-line-tools/share/JLink_Linux_V794e_arm64.deb --fix-broken
```

3. Download the UWB apps repository.

```no-highlight
git clone git@github.com:j-peller/uwb-apps.git
cd uwb-apps
```

4. Running the ```newt upgrade``` command downloads the apache-mynewt-core, decawave-uwb-core, decawave-uwb-dwXXXX driver(s) repo, and mynewt-timescale-lib packages, these are dependent repos of the decawave-uwb-apps project and are automatically checked-out by the newt tools.

```no-highlight
newt upgrade
```

To see if you have access to other driver repos, run the setup.sh
script under repos/decawave-uwb-core like so:

```
    repos/decawave-uwb-core/setup.sh
    newt upgrade
```

Apply any patches to apache-mynewt-core:

```
cd repos/apache-mynewt-core/
git apply ../decawave-uwb-core/patches/apache-mynewt-core/mynewt_1_7_0_*
git apply ../decawave-uwb-core/patches/apache-mynewt-core/mynewt_1_8_*
cd -
```

5. Erase the default flash image that shipped with the DWM1001.

```
    $ nrfjprog -f NRF52 -e
```

6. Install the correct version of ARM Toolchain

Download for Linux 32-bit: https://developer.arm.com/downloads/-/gnu-rm/5-2016-q3-update

```
bzip2 -d gcc-arm-none-eabi-5_4-2016q3-20160926-linux.tar.bz2
tar -xvf gcc-arm-none-eabi-5_4-2016q3-20160926-linux.tar

mv gcc-arm-none-eabi-5_4-2016q3/ /opt/

export PATH=/opt/gcc-arm-none-eabi-5_4-2016q3/bin:$PATH
source ~/.bashrc

/* check if working */
arm-none-eabi-gcc --version
```

If 32bit not supported on your current system:

```
dpkg --add-architecture i386

echo "deb http://security.ubuntu.com/ubuntu focal-security main universe" | sudo tee /etc/apt/sources.list.d/focal-security.list

apt update
apt install libc6:i386 libstdc++6:i386 libncurses5:i386
```

### Build bootloader

Builds bootloader applicaiton for the DWM1001 target. Bootloader should be flashed first for each DWM1001 module regardless of the role:

```no-highlight
newt target create dwm1001_boot
newt target set dwm1001_boot app=@mcuboot/boot/mynewt
newt target set dwm1001_boot bsp=@decawave-uwb-core/hw/bsp/dwm1001
newt target set dwm1001_boot build_profile=optimized
newt build dwm1001_boot

# Execute for _each_ DWM1001 module
newt load dwm1001_boot
```

### Master module

Only one module among anchors which acts as a master is allowed per network. In this configuration it is possible to have 4 anchors minimum and up to 8 modules:

```no-highlight
# dont forget to flash the boot loader first
newt load dwm1001_boot

# now start flashing twr firmware 
newt target create nrng_master
newt target set nrng_master app=apps/twr_nranges_tdma
newt target set nrng_master bsp=@decawave-uwb-core/hw/bsp/dwm1001
newt target set nrng_master build_profile=optimized

# Disable additional debug output
newt target amend nrng_master syscfg=NRNG_VERBOSE=0:SURVEY_ENABLED=0

# Disable RTT enable UART
newt target amend nrng_master syscfg=CONSOLE_RTT=0:CONSOLE_UART=1

# Configure number of nodes and frames
newt target amend nrng_master syscfg=NRNG_NFRAMES=16:NRNG_NNODES=8:NRNG_NTAGS=4:NODE_START_SLOT_ID=0:NODE_END_SLOT_ID=7

# Configure role
newt target amend nrng_master syscfg=PANMASTER_ISSUER=1:UWB_TRANSPORT_ROLE=0

newt create-image nrng_master 1.0.0

# this flashes the image onto the dwm1001, ensure usb connectivity
newt load nrng_master
```

### Slave modules

All other non-master anchors should be configured as slaves:

```no-highlight
# dont forget to flash the boot loader first
newt load dwm1001_boot

# now start flashing twr firmware 
newt target create nrng_slave
newt target set nrng_slave app=apps/twr_nranges_tdma
newt target set nrng_slave bsp=@decawave-uwb-core/hw/bsp/dwm1001
newt target set nrng_slave build_profile=optimized

# Disable additional debug output
newt target amend nrng_slave syscfg=NRNG_VERBOSE=0:SURVEY_ENABLED=0

# Disable RTT and UART
newt target amend nrng_slave syscfg=CONSOLE_RTT=0:CONSOLE_UART=1

# Configure number of nodes and frames
newt target amend nrng_slave syscfg=NRNG_NFRAMES=16:NRNG_NNODES=8:NRNG_NTAGS=4:NODE_START_SLOT_ID=0:NODE_END_SLOT_ID=7

# Configure role
newt target amend nrng_slave syscfg=NRANGES_ANCHOR=1:UWB_TRANSPORT_ROLE=0

newt create-image nrng_slave 1.0.0

# this flashes the image onto the dwm1001, ensure usb connectivity
newt load nrng_slave
```

### Tag module

Only one module acts as a tag and is mounted on the drone, connected to the Raspberry Pi 5 via UART

```no-highlight
# dont forget to flash the boot loader first
newt load dwm1001_boot

# now start flashing twr firmware 
newt target create nrng_tag
newt target set nrng_tag app=apps/twr_nranges_tdma
newt target set nrng_tag bsp=@decawave-uwb-core/hw/bsp/dwm1001
newt target set nrng_tag build_profile=optimized

# Disable additional debug output
newt target amend nrng_tag syscfg=NRNG_VERBOSE=0:SURVEY_ENABLED=0

# Disable RTT and UART
newt target amend nrng_tag syscfg=CONSOLE_RTT=0:CONSOLE_UART=1

# Configure number of nodes and frames
newt target amend nrng_tag syscfg=NRNG_NFRAMES=16:NRNG_NNODES=8:NRNG_NTAGS=4:NODE_START_SLOT_ID=0:NODE_END_SLOT_ID=7

# Configure role
newt target amend nrng_tag syscfg=UWB_TRANSPORT_ROLE=1

newt create-image nrng_tag 1.0.0

# this flashes the image onto the dwm1001, ensure usb connectivity
newt load nrng_tag
```

**!!!!! Don't forget to load bootloader for each module !!!!!**

## Test your Anchor and Tag
1. Connect the Anchor to a power source 

2. Connect the Tag to your PC and access it via UART (typically /dev/ttyACMX or /dev/ttyUSBX)

Use either putty or minicom...
```
minicom -D /dev/ttyACM0 -b 460800
```

This launches the interactive command shell of the DWM1001 firmware running on the Tag. Once initialized, the Tag should automatically detect the powered Anchor and begin outputting measured distances to the console.

If you disconnect the Anchor, the output will stop. You can power on additional Anchors at any time—each will be automatically recognized by the Tag and included in the distance measurements.
