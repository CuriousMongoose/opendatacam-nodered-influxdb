# Nvidia Jetson and OpenDataCam to Explore Computer Vision and IoT Analytics

Using a [Nvidia Jetson Nano](https://developer.nvidia.com/embedded/jetson-nano-developer-kit) running [OpenDataCam](https://opendatacam.github.io/opendatacam/), it simple count people and vehicles on almost any video feed, and send the data to a [InfluxDB Cloud](https://www.influxdata.com/products/influxdb-cloud/) instance using [Node-RED](https://nodered.org/) via a cellular connection for remote deployments.

Complete details and instructions for this project can be found on the [Hologram blog](https://www.hologram.io/blog-categories). This page only supplies supporting information and code.

## Node-RED flow

To use *nodered-flow.json*, you need to install additional nodes  
`npm install node-red-contrib-stackhero-influxdb-v2`

## Cellular modem setup

The D-Link DWM-222 that I used for this project required some additional setup, and you might find this information relevant for other modems as well.

Dongles intended for Windows machines will start by showing as a mass storage device for driver download, and then change to modem mode when it detects a driver. To force this switch on Linux, we use [USB_ModeSwitch](https://www.draisberghof.de/usb_modeswitch/). There were no configuration files for my modem, so I had to create a custom config file, using the vendor and product IDs that I was able to get by running `lsusb`.

```txt
Bus 001 Device 005: ID 2001:ac01 D-Link Corp.
```

Create the config file  
`sudo nano /etc/usb_modeswitch.d/2001\:ac01`

Copy the following contents into the file.

```txt
# D-Link DWM-222 A2
DefaultVendor=0x2001
DefaultProduct=0xac01
TargetVendor=0x2001
TargetProduct=0x7e3d
StandardEject=1
OptionMode=0
```

Open the udev rules file for USB_ModeSwitch
`sudo nano /lib/udev/rules.d/40-usb_modeswitch.rules`

Add the following entry

```txt
# D-Link DWM-222 A2
ATTR{idVendor}=="2001", ATTR{idProduct}=="ac01", RUN+="usb_modeswitch '/%k'"
```

Restart the Jetson, and run `lsusb` to check if the modeswitch worked. The result should be

```bash
Bus 001 Device 005: ID 2001:7e3d D-Link Corp.
```

Check if ModemManager detects the modem using `mmcli -L`

If not the modem was probably not assigned a serial ID. This can be confirmed by running `ls /dev/ttyUSB*`, which should return nothing.

Assign the dongle a ID

```bash
sudo modprobe option;sudo sh -c "echo 2001 7e3d > /sys/bus/usb-serial/drivers/option1/new_id"
```

Running `ls /dev/ttyUSB*` should now return `ttUSB0` to `ttyUSB4`

Running `mmcli -L` should now show the modem

The rest of the [Network Manager](https://docs.ubuntu.com/core/en/stacks/network/network-manager/docs/configure-cellular-connections) setup instructions are in the [Hologram blog post]()

## [OpenDataCam USB Web Cam Setup](https://github.com/opendatacam/opendatacam/blob/master/documentation/CONFIG.md)
If your USB web cam is not detected by OpenDataCam, after updating config file `sudo nano ~/opendatacam/config.json`

```json
"VIDEO_INPUT": "usbcam"
```

1. Verify if you have an usbcam detected

   ```bash
   ls /dev/video*
   # Output should be: /dev/video0
   ```

2. Check what resolution and framerate your webcam supports, by running `v4l2-ctl --list-formats-ext`. In my case it was 800x600 at 7.5 FPS (15/2 FPS).

3. Update the opendatacam/config.json file as follows

   ```json
   "VIDEO_INPUTS_PARAMS": {
     "usbcam": "v4l2src device=/dev/video0 ! video/x-raw, framerate=15/2, width=800, height=600 ! videoconvert ! appsink"
   }
   ```

4. (Optional) If your device is on `video1` or `video2` instead of default `video0`, change `VIDEO_INPUTS_PARAMS > usbcam` to your video device, for example if /dev/video1

## Additional Information Sources

- [JetsonHacks](https://www.jetsonhacks.com/): Projects for the Jetson series of devices
- [Jetson Setup note by Raspberry Pi Valley](https://raspberry-valley.azurewebsites.net/NVIDIA-Jetson-Nano/): A collection of notes on setting up various packages on the Jetson Nano
- [Object Detection in 10 lines of code](https://news.developer.nvidia.com/realtime-object-detection-in-10-lines-of-python-on-jetson-nano/): A cool little tutorial from Nvidia
- [Hello AI World](https://github.com/dusty-nv/jetson-inference): A more comprehensive series of tutorials from Nvidia