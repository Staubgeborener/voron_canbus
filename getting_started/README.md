## Getting Started

# can0 file, CAN Speeds, and Transmit Queue Length

In order to dictate the speed at which your CAN network runs at you will need to create (or modify) a "can0" file on your Pi. This is what will tell linux "Hey, you now have a new network interface called can0 that you can send CAN traffic over". The approach needed here heavily depends on the network stack of your Pi. Raspbian and older version of Debian typically use ifupdown, but newer versions of non-raspbian Debian use netplan, which by default uses systemd-networkd under the hood.

To test if your system uses networkd, try running `networkctl`. If the command is not available, gives an error or no link output, you're probably still using ifupdown. If it does give you meaningful output (typically at least an `ether` and `wlan` link), you're using systemd-networkd. To be safe it's easiest to just follow both the "ifupdown" **and** the "systemd-networkd" instructions. You can have both set up and your Pi will just use the configuration that is relevant to your system.

To set everything up, SSH into your pi and run the commands needed for your network setup:

## ifupdown
  ```
  sudo nano /etc/network/interfaces.d/can0
  ```
  ![image](https://user-images.githubusercontent.com/124253477/221327674-fad20589-1a5b-4d68-b2d9-2596553f64ab.png)

This will open (or create if it doesn't exist) a file called 'can0' in which you need to enter the following information:
  ```
  allow-hotplug can0
  iface can0 can static
    bitrate 1000000
    up ip link set can0 txqueuelen 1024
  ```

![image](https://user-images.githubusercontent.com/124253477/221378593-9a0fcdb5-082c-454e-94bd-08a6dc449d34.png)

Press Ctrl+X to save the can0 file.

The "allow-hotplug" helps the CAN nodes come back online when doing a "firmware_restart" within Klipper.
"bitrate" dictates the speed at which your CAN network runs at. Kevin O'Connor (of Klipper fame) recommends a 1M speed for this to help with high-bandwidth and timing-critical operations (ADXL Shaper calibration and mesh probing for example).
To complement a high bitrate, setting a high transmit queue length "txqueuelen" of 1024 helps minimise "Timer too close" errors.

Once the can0 file is created just reboot the Pi with a `sudo reboot` and move on to the next step.

## systemd-networkd (netplan)
  ```
  sudo nano /etc/systemd/network/10-can.link
  ```
This will open (or create if it doesn't exist) a file called `10-can.link` in which you need to enter the following information:
  ```
  [Match]
  Type=can

  [Link]
  TransmitQueueLength=1024
  ```
Press Ctrl+X to save the file.

To set the bitrate, we need to create another file in the same directory:
  ```
  sudo nano /etc/systemd/network/25-can.network
  ```
This creates a network file, that networkd will then use to automatically set up the network with. Because of how networkd works, "hotplugging" is baked in.
Enter the following in the network-file and close it with Ctrl+X:
  ```
  [Match]
  Name=can*

  [CAN]
  BitRate=1M
  ```
  
  

# Next Step

Now that the can0 interface files are set up, you need to choose how you are going to connect your Pi to the CANBus network.

To actually create a CAN network in your system, your Pi needs some sort of device to act as a CAN adapter (think of it like a USB network card, or USB wifi dongle). The simplest plug-and-play option is to use a dedicated USB to Can device such as the BigTreeTech U2C, Mellow Fly UTOC, Fysetc UCAN, etc. (other devices exist as well). The second "cheaper" option is to actually utilise your printer mainboard (ie. Octopus or Spider board) to function as a usb-can bridge for klipper. We'll cover both options, but you only need to choose one.




# Dedicated USB CAN device

**IF YOU HAVE A BTT U2C V2.1 THEN PLEASE FLASH IT WITH THE LATEST VERSION OF V2 FIRMWARE FROM THE GITHUB AS THE SHIPPED FIRMWARE MAY HAVE BUGS https://github.com/Esoterical/voron_canbus/tree/main/can_adapter/BigTreeTech%20U2C%20v2.1**

If you want to use a dedicated USB CAN devcice, then it should be as  simple as plugging it in to your Pi via USB. ***You shouldn't have to flash anything to this U2C/UTOC/etc device first, they are meant to come pre-installed with the necessary firmware. They do NOT run Klipper***. You can test if it is working by running an `lsusb` command (from an SSH terminal to your pi). Most common USB CAN devices will show up as a "Geschwister Schneider CAN adapter" when they are working properly (though some may simply show as an "OpenMoko, Inc" device):

![image](https://user-images.githubusercontent.com/124253477/221329262-d8758abd-62cb-4bb6-9b4f-7bc0f615b5de.png)

![image](https://user-images.githubusercontent.com/124253477/222042688-10fa6fdb-8c0a-4142-8c40-0d93ef4fc4bd.png)


A better check is by running an 'interface config' command `ifconfig`. If the USB CAN device is up and happy (and you have created the can0 file above) then you will see a can0 interface:

![image](https://user-images.githubusercontent.com/124253477/221329326-efa1437e-839d-4a6b-9648-89412791b819.png)

**A note on edge cases**

If you plug in your USB CAN adapter and you *don't* see the expected results from an `lsusb` or `ifconfig`, then the firmware on your device may have issues. If this is the case then it's worth going to the Github page of your device as they usually have the stock firmware and flashing instructions there.

**A note on the note**

The BTT U2C V2.1 was released with bad firmware which although would show up to the above tests it would make issues show up down the line. If you have a v2.1 of the U2C then please follow the instructions here: https://github.com/Esoterical/voron_canbus/tree/main/can_adapter/BigTreeTech%20U2C%20v2.1

# Klipper USB to CAN bus bridge

The second way of setting up a CAN network is to use the printer mainboard itself as a CAN adapter.

**If you are using a dedicated CAN adapter as above then you don't need this step. Your mainboard will be flashed the same as any other "normal" klipper install**

This is acheived through Klippers "USB-CAN-Bridge mode". In order for this to work you need to have a compatible MCU on the mainboard (A lot of the popular STM32 chips works, as well as the RP2040), and either a dedicated "CAN" port on the motherboard or at least a way of accessing the CAN pins that you configure for klipper.

Some mainboards (like the BTT Octopus) have a CAN Transceiver built in so they will output CAN signals directly from a dedicated port (the Octopus has an RJ11 port for this purpose). Other compatible boards may have a port on their board labelled as CAN but only output serial (Tx Rx) signals. These boards can still be run as USB-CAN-Bridge mode but will require an additional CAN Transceiver module (such as the SN65HVD230). These can be cheaply purchased from Amazon or eBay or AliExpress. Other boards may yet not have any dedicated CAN port, but still have a compatible MCU and have compatible CAN pins that you can access (the SKR Mini E3 V3 can be run in USB-CAN-Bridge mode if you use the PB8/PB9 pins on the EXP1 header that is normally used for an LCD screen).

More specific instructions refer to https://github.com/Esoterical/voron_canbus/tree/main/mainboard_flashing

Once you have klipper firmware flashed to your mainboard, with the USB-CAN-Bridge mode enabled, it should show up to your Pi as a "Geschwister Schneider CAN adapter" if you run an `lsusb`

![image](https://user-images.githubusercontent.com/124253477/221329262-d8758abd-62cb-4bb6-9b4f-7bc0f615b5de.png)

If you run an `ifconfig` command you should also see a can0 interface.

![image](https://user-images.githubusercontent.com/124253477/221329326-efa1437e-839d-4a6b-9648-89412791b819.png)

The takeaway is that if you go down the mainboard USB-CAN-Bridge route, then you *need* to have klipper firmware flashed to the mainboard before attempting any further CAN installs/troubleshooting.