---
layout: post
title:  "How to get the Altera USB Blaster to work in Ubuntu 20.04"
date:   2020-09-23 14:08:44 -0700
categories: linux fpga
---

# Background

I just got boards back for a new project using the [Intel Max 10 FPGA](https://www.intel.com/content/www/us/en/products/programmable/fpga/max-10.html). When attempting to program the device using an Altera USB Blaster [knockoff](https://www.amazon.com/gp/product/B00IRODADK/ref=ppx_yo_dt_b_asin_title_o06_s00?ie=UTF8&psc=1) with Quartus Prime I got an error stating that it was "unable to scan device chain". This cryptic message had me checking all of the electrical connections on my board thinking there was something wrong with the JTAG port or the FPGA itself. When I eventually Googled this error message I discovered that this is not a hardware problem but an issue with Ubuntu and Quartus. See below for how I resolved this.

# Create udev rule

Create a new file **/etc/udev/rules.d/51-usbblaster.rules** and put the following in it:

{% highlight conf %}
# USB Blaster
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6001", MODE="0666", NAME="bus/usb/$env{BUSNUM}/$env{DEVNUM}", RUN+="/bin/chmod 0666 %c"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6002", MODE="0666", NAME="bus/usb/$env{BUSNUM}/$env{DEVNUM}", RUN+="/bin/chmod 0666 %c"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6003", MODE="0666", NAME="bus/usb/$env{BUSNUM}/$env{DEVNUM}", RUN+="/bin/chmod 0666 %c"

# USB Blaster II
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6010", MODE="0666", NAME="bus/usb/$env{BUSNUM}/$env{DEVNUM}", RUN+="/bin/chmod 0666 %c"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6810", MODE="0666", NAME="bus/usb/$env{BUSNUM}/$env{DEVNUM}", RUN+="/bin/chmod 0666 %c"

{% endhighlight %}

# Install libudev and create a symbolic link to it
{% highlight console %}
$ sudo apt-get install libudev-dev
$ sudo ln -sf /lib/x86_64-linux-gnu/libudev.so.1 /lib/x86_64-linux-gnu/libudev.so.0
{% endhighlight %}

After the two steps above all was good!