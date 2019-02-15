---
layout: post
title: Sane management of multiple RTL-SDR dongles in Linux
tags: 
  - SDR
  - Linux
  - Other Projects
categories:
  - Other Projects
---

I was looking to utilize a couple RTL dongles to monitor two ISM band frequencies commonly used in LoRa without buying an SDR with wide enough bandwidth to cover both ranges.  I pretty quickly ran into issues with how SpyServer and rtl_tcp enumerate devices, which appears to be based mostly upon the order in which each device had been plugged in.  With some work, I think I've come upon a flexible and secure solution to handle an arbitrary number of dongles on one system while maintaining deterministic control of each device.  This means I can label an individual dongle, connect it to the desired antenna, and then connect to that dongle on the assigned TCP port every time, without regard to the order in which things have been plugged in.

All of these steps have been performed on a fresh installation of Raspbian Stretch.

# HOWTO: Sane management of multiple RTL-SDR dongles in Linux

## Prepare the system

```bash
# get the latest packages and apply any updates
sudo apt-get update && sudo apt-get upgrade
# install required packages
sudo apt install rtl-sdr librtlsdr-dev -y
# get the latest udev rules for common dongles
sudo wget -O /etc/udev/rules.d/20-rtlsdr.rules https://raw.githubusercontent.com/osmocom/rtl-sdr/master/rtl-sdr.rules
# reboot to apply udev rules
sudo reboot
```

## Create unprivileged service user
Here we create a system user "rtltcp" with no login privileges or shell to help protect against shell escapes if opening this service to the internet

```bash
sudo adduser --system rtltcp
sudo addgroup rtltcp
sudo usermod -aG dialout,rtltcp -a rtltcp
```

## Define device serial numbers to match TCP port

We'll define a serial number for each device to match the TCP port. I've applied a physical label to each dongle for easy identification, so I can plug in the desired antenna to the correct device and then connect to the TCP port written on the label.

```bash
# First, run rtl_eeprom to display installed devices
rtl_eeprom
# Then assign the desired SN to each device.  We want the SN to line up w/ the TCP port, so pick a valid value.
# Here I'm setting up two dongles to present on ports 1230 and 1231.  Change these lines to match your requirements!
rtl_eeprom -d0 -s 1230
rtl_eeprom -d1 -s 1231
```

## Create script to identify dongles by SN
rtl_tcp identifies devices by device number, which is non-deterministic.  The script below will accept a serial number as an argument, and return the device ID assigned to the provided serial number.

```bash
sudo mkdir /opt/rtltcp
sudo tee /opt/rtltcp/rtlsn2dev.sh << 'EOF'
#!/bin/bash
# usage: ./rtlsn2dev.sh [serial number]
# returns: device ID
if [ $1 ]
then
  rtl_num_devices=$(rtl_eeprom 2>&1 >/dev/null | grep "Found [0-9][0-9]*" | sed -E 's/.*([0-9]+).*/\1/')
  if [ $rtl_num_devices ]
  then
    for i in $(seq 1 $rtl_num_devices);
    do
      rtl_device=$((i-1))
      rtl_serial=$(rtl_eeprom -d$rtl_device 2>&1 >/dev/null | grep "Serial number\:" | sed -E 's/Serial number:[[:blank:]]+//')
      if [ "$1" == "$rtl_serial" ]
      then
        echo $rtl_device
      fi
    done
  fi
fi
EOF
sudo chmod +x /opt/rtltcp/rtlsn2dev.sh
sudo chown -R rtltcp:rtltcp /opt/rtltcp
```

## Create systemd service unit file
Now we'll make a systemd service template which we'll then use for each instance we plan to run.
```bash
sudo tee /etc/systemd/system/rtltcp@.service << 'EOF'
[Unit]
Description=rtl_tcp radio streaming service
After=network.target

[Install]
WantedBy=multi-user.target

[Service]
Type=simple
User=%p
Restart=always
WorkingDirectory=/opt/rtltcp
ExecStart=/bin/sh -c "/usr/bin/rtl_tcp -d $(/opt/rtltcp/rtlsn2dev.sh %i) -a 0.0.0.0 -p %i"
EOF

# Now reload systemd, create a couple instances of this service, and start them
sudo systemctl --system daemon-reload
sudo systemctl enable rtltcp@1230.service
sudo systemctl enable rtltcp@1231.service
sudo systemctl start rtltcp@1230.service
sudo systemctl start rtltcp@1231.service
```

And that's it!  Now you have a couple services which will run at startup and can be controlled/monitored via conventional methods for linux system administration, and which will "stick" to the assigned dongle.