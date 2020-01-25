# ihsctrl

**ihsctrl** - a package of bash scripts to control selected IKEA Home
smart (aka “TRÅDFRI”) devices via their network gateway.

## Limitations

The included scripts are intended to turn the following individual
devices *on* or *off*: 

* [IKEA TRÅDFRI Wireless control outlet
  303.561.69](https://www.ikea.com/ca/en/p/tradfri-wireless-control-outlet-30356169/)
  
* [IKEA TRÅDFRI LED bulb E26 600 lumen
  904.086.17](https://www.ikea.com/ca/en/p/tradfri-led-bulb-e26-600-lumen-wireless-dimmable-color-and-white-spectrum-opal-white-90408617/)
  
No dimming or colour change is supported. Other lighting devices may
work but are untried. Smart blinds and music control almost certainly
won't.

*ihsctrl* also only supports one gateway per installation.

In order to keep the code simple and to avoid implementing a server model, commands sent simultaneously to the gateway have undefined results: one command may succeed, but more likely, all simultaneous commands may fail. This can be a confusing issue when using *ihsctrl* from cron, as jobs triggered from separate crontab lines may collide and fail.

## Installation

This software was developed, (minimally) tested and deployed on
current Debian-based Linux systems. They are essentially wrappers
around [libcoap2](https://libcoap.net/)'s `coap-client-gnutls` with
JSON massaged using [jq](https://stedolan.github.io/jq/). These should
be run as a regular user, and an ideal place for them to live is in the
user's *~/bin* folder.

### Installation on Ubuntu / Raspbian

    sudo apt install libcoap2-bin jq

If the *~/bin* folder doesn't exist, make it and temporarily add it to
your PATH:

    mkdir ~/bin
    PATH=${PATH}:~/bin
	
*~/bin* is automatically added to a user's path if it exists, so no
modification of profiles is required.

Clone this repository and change to its directory:

    git clone https://github.com/scruss/ihsctrl.git
    cd ihsctrl

Now copy the wrapper scripts to *~/bin* and ensure they are
executable:

    cp ihs* ~/bin
    chmod +x ~/bin/ihs*
	
You can now set up the connection using `ihsinit`. To do this, you
should have:

1. the local IP address or host name of the [IKEA TRÅDFRI Gateway
   003.378.13](https://www.ikea.com/ca/en/p/tradfri-gateway-white-00337813/)
   attached to your network. This might show up via `nmap` or your
   router's network map under a name like
   *TRADFRI-Gateway-xxxxxxxxxxxx* and can be reached at the
   corresponding mDNS .local address
   
2. the 16-character Security Code printed on the back of your IKEA
   TRÅDFRI Gateway, just under the QR code. This code is never stored
   on your system by the *ihsctrl* scripts; instead, a shared key is
   returned by the gateway and this is used for all future
   communication
   
3. a simple (but unique for this network) user name. The content isn't
   important, but attempting to register a name already in use from
   another computer on your network will fail
   
4. (Optionally) a working gateway with at least one verified/paired device
   already on your network. This will give you something to test the
   `ihsswitch` command with.

## Usage

### ihsinit

    ihsinit gateway_address security_code username
	
Example:

    ihsinit 192.168.1.112 'xW406zAduzkXyq3k' fred
	
If the gateway is found at *gateway_address*, the two arguments
*security_code* and *username* are hashed inside the gateway to
produce a shared key. This shared key, along with the address and user
details, are stored in *~/.config/ihsctrl/ihsconf.json*.

Credentials are valid for some time (60 days?) and are refreshed every
time you access the gateway. If they expire, you'll need to delete the
configuration file and run `ihsinit` again.

If the configuration is successful, `ihsinit` attempts to run
`ihsinfo` to list known devices. If it is unsuccessful, it suggests
some issues to remedy the fault.

### ihsinfo

    ihsinfo
	
Takes no arguments, but lists devices known to the gateway. Mostly
useful for choosing a device ID to use with `ihsswitch`

Sample output:

    id     name                     type                             status
    =====  =======================  ===============================  ======
    65537  front light              TRADFRI control outlet           1     
    65544  desk lamp                TRADFRI bulb E26 CWS opal 600lm  1     
    65545  TRADFRI remote control   TRADFRI remote control           n/a   
    65546  front rgb                TRADFRI control outlet           1     
    65547  fish                     TRADFRI control outlet           1     
    65548  TRADFRI signal repeater  TRADFRI signal repeater          n/a   
    65549  ender3                   TRADFRI control outlet           0  

### ihsswitch

    ihsswitch id (0|1)
	
Example:

    ihsswitch 65544 0
	
would turn the IKEA TRÅDFRI bulb (id 65544, named ‘desk lamp’ above)
off. Replacing the last argument with 1 would turn it back on.

The (0|1) final argument is not checked, but using anything other than
0 or 1 will not do anything useful.

### ihsstatus

    ihsstatus id
    
Example:

    ihsstatus 65544
    
Returns 0 or 1, depending on whether the device is on or off. If the device has no on or off status, returns nothing.

### ihstoggle

    ihstoggle id
    
Example:

    ihstoggle 65544
    
Toggles the power state of a device, or does nothing if the device returns no status. If you think this script feeds the inverted output of `ihsstatus` into `ihsswitch`, you're darn right it does!

### ihslist

The least useful of the tools, it is used internally by `ihsinfo` and
`ihsswitch` to list known devices. It takes no arguments:

    ihslist
	
Sample output:

    65537
    65544
    65545
    65546
    65547
    65548
    65549

## Use with cron

This suite is intended for use in user cron jobs for simple daily
lighting tasks. A sample `crontab -l` listing is as follows, with all
times local:

    PATH=/bin:/usr/bin:/home/pi/bin
    # m h  dom mon dow   command
    # front light: 65537
     30 6   *   *   1-5   ihsswitch 65537 1
     30 7   *   *   6,7   ihsswitch 65537 1
     15 8	*   *   1-5   ihsswitch 65537 0
     15 9   *   *   6,7   ihsswitch 65537 0
     10 10  *   *   *     sunwait sun down -1:00:00 43.7N 79.3W; ihsswitch 65537 1
     45 22  *   *   *     ihsswitch 65537 0

* `cron` runs under a very simple shell (typically `/bin/sh`) with a
  very limited command search path (initially */bin:/usr/bin*), so the
  path to the *ihsctrl* programs must be stated (line 1). Note that
  `cron` does **not** expand variable definitions such as `$HOME/bin`.
  
* at 06:30 on weekdays and 07:30 on weekends, turn on the front light
  (ID 65537) - lines 4 & 5.
  
* at 08:15 on weekdays and 09:15 on weekends, turn off the front
  light - lines 6 & 7. 
  
* at 10:10 every morning, launch
  [sunwait](https://github.com/risacher/sunwait) to wait and turn on the front
  light an hour before sunset at a location in the east end of
  Toronto - line 8.
  
* at 22:45 every day, turn the front light off - line 9.

**NB**: The author uses an older version of sunwait
([sunwait-20041208.tar.gz](https://web.archive.org/web/20151016135925/http://www.risacher.org/sunwait/sunwait-20041208.tar.gz))
and the current version may have different calling syntax.

**IMPORTANT** *ihsctrl* does not support simultaneous switching of multiple devices, as noted in **Limitations**. Please check your crontab for jobs that might trigger at exactly the same time: they are *highly unlikely* to work as expected. Workarounds for this include:

* combining command fields into the one crontab line: `* * * * * ihsswitch … ; ihsswitch … `, etc.

* delaying commands by a small amount: if one job triggers at 07:30, say, make the competing job run at 07:31.

This may be an insurmountable problem for some use cases.

## Files

*~/.config/ihsctrl/ihsconf.json* stores the access credentials for the
gateway.

## Observations

* On first connecting a gateway and devices, there can be a
  considerable delay (up to a day) before they all have updated
  firmware. If you have problems getting devices to appear as they
  should in the IKEA Home smart app, let them sit connected until they
  all show current firmware, then remove them, then re-add them all
  via the app. The app is not required to control devices via
  *ihsctrl*, and a new device can be added by pairing through a
  [remote](https://www.ikea.com/ca/en/p/tradfri-remote-control-00443130/)
  without the app. This won't allow the device to be renamed or added
  to a room, though.
  
* Control outlets briefly turn off and on again when their firmware
  updates. They do this relatively seldom, but can do it at any
  time. Consequently, these outlets *may* not be ideal for controlling
  devices such as 3D printers.
  
* The USB port on the [signal
  repeater](https://www.ikea.com/ca/en/p/tradfri-signal-repeater-30400407/)
  seems to provide adequate current to drive a [Raspberry Pi Zero
  W](https://www.raspberrypi.org/products/raspberry-pi-zero-w/). The
  author uses this as his (tiny, adequate, cheap) lighting control computer.

## Rationale

1. These scripts are intended to replace the author's very old
   [X10](https://www.x10.com/) lighting control
   [system](https://scruss.com/blog/tag/x10/) that has been running on
   a Raspberry Pi since 2012. The mechanical switches in the (old,
   surplus) X10 outlets I used were becoming unreliable. Replacing
   these would have been as expensive as buying a whole new system.
   
2. The private security code printed on the back of the IKEA TRÅDFRI
   gateway is never stored: only the derived shared key is kept, and
   it's useless on its own.
   
3. IKEA TRÅDFRI devices cannot be accessed over the Internet and do
   not seem to share any information over the Internet. The gateway
   does check for firmware updates for attached devices, and will
   download those and do OTA updates (slowly) via ZigBee.
   
4. ihsctrl scripts remember all the authentication details: you should
   never have to remember the LAN address of your gateway, the shared
   key or the user ID.
   
5. The scripts run with standard user privileges from the *~/bin*
   folder, and are intended for use in cron jobs and other
   low-complexity tasks.
   
6. Scripts and howtos on the web from when this hardware was launched (2017-2018) rely on building a custom version of the libcoap2 tools. Those instructions no longer work.

## Author

Stewart Russell — scruss.com — 2020-01

## Licence

Copyright © 2020 Stewart Russell <scruss@scruss.com>
This work is free. You can redistribute it and/or modify it under the
terms of the Do What The Fuck You Want To Public Licence, Version 2,
as published by Sam Hocevar. See the COPYING file for more details.

    /* This program is free software. It comes without any warranty, to
     * the extent permitted by applicable law. You can redistribute it
     * and/or modify it under the terms of the Do What The Fuck You Want
     * To Public Licence, Version 2, as published by Sam Hocevar. See
     * http://www.wtfpl.net/ for more details. */

## Acknowledgements

* This is not an officially endorsed product. Use it entirely at your
  own risk.

* “IKEA”, and all related names, logos, product and service names,
  designs and slogans are trademarks of Inter IKEA Systems B.V. 
  
* IKEA has noted to developers that the method of access used here may
  go away at any time.
  
* Details of the protocol used was learned from [glenndehaan /
ikea-tradfri-coap-docs](https://github.com/glenndehaan/ikea-tradfri-coap-docs)

* Details of how to turn devices on and off was learned from
  [Pimoroni](https://pimoroni.com/)'s tutorial “[Controlling IKEA
  Trådfri Lights from your
  Pi](https://learn.pimoroni.com/tutorial/sandyj/controlling-ikea-tradfri-lights-from-your-pi)” 

