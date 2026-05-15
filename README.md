# Jetson nano 4gb

This repo contains builds of armbian for a jetson nano 4gb and notes on getting them working. I have an a02 board.

My jetson has a dead sd card reader, so the instructions at jetson-original-image-usb.md let me boot from usb with no sd card.

Next, I read that arhcived builds of armbian 23.8.1 boot on it. The instructions at jetson-armbian-23.md show how to get it working, because it didn't work straight away for me, maybe becuase I'm using a usb drive. HDMI was working when I flashed a minimal build, and a shell came up on the monitor.

The instructions at jetson-armbian-26.md show how to get armbian 26 trixie booted. This is headless only, hdmi does not work.

The bootlogs folder contains uart logs of attempting to boot an armbian 24 build. Builds are in the releases of this repo and were made with the armbian/build action.