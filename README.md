# NUC8IxBEx Hackintosh
This is a quick and dirty repo for Intel NUC 8th gen computers. It should work on all the Coffee Lake ones. I've used various sources to get to this point and did quite some testing. It should leave you with a stable and reliable build but as always, these things are never really finished. While it should work on older macOS versions, I've done all building and testing on Catalina and Big Sur.

## Details
* Works with macOS *Catalina* and *Big Sur*[\*](#big-sur)
* OpenCore bootloader with the following kexts:
  - Lilu
  - VirtualSMC
  - WhateverGreen
  - AppleALC
  - IntelMausi
  - NVMeFix
  - CPUFriend
  - OpenIntelWireless kexts for Intel bluetooth and wifi
  - FakePCIID (without this audio over hdmi only works when re-plugging the cable)
  
## Index
* [Installation](#installation)
* [Post install](#post-install)
* [Big Sur](#big-sur)
* [Apple and 3rd party wifi/bt](#apple3rd-party-bluetooth-and-wifi)
* [ThunderBolt](#thunderbolt)
* [Intel wifi/bt](#intel-bluetooth-and-wifi)
* [Native bt dongle](#natively-supported-bluetooth-dongle)
* [What doesn't work?](#not-workinguntested)
* [Performance, power and noise](#performance-power-and-noise)
  - [Noise](#noise)
  - [Passive cooling](#passive-cooling)
* [Todo](#todo)
* [Credits](#credits)
  
## Installation
+ Update to latest (0085) BIOS -> load BIOS defaults -> click advanced and change;
```
Devices -> USB -> Port Device Charging Mode: off
Security -> Thunderbolt Security Level: Legacy Mode
Power -> Wake on LAN from S4/S5: Stay Off
Boot -> Boot Configuration -> Network Boot: Disable
Boot -> Secure Boot -> Disable
```
+ Download macOS form the App Store and create a USB installer with *[createinstallmedia](https://support.apple.com/en-us/HT201372)* on macOS (real mac/hack or vm) or use [gibMacOS](https://github.com/corpnewt/gibMacOS)\*
+ Download [this repo](https://github.com/zearp/Nucintosh/archive/master.zip) and extract the EFI folder from the archive
+ Edit config.plist with [ProperTree](https://github.com/corpnewt/ProperTree) and change the following fields;
```
PlatformInfo -> Generic -> MLB
PlatformInfo -> Generic -> ROM
PlatformInfo -> Generic -> SystemSerialNumber
PlatformInfo -> Generic -> SystemUUID
```
Generate new serials with [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS). The ROM value is your ethernet (en0) mac address ([more info](https://dortania.github.io/OpenCore-Post-Install/universal/iservices.html#fixing-en0)).
+ Copy the EFI folder to the EFI partition on the USB installer
+ Install macOS

\* Installers made with GibMacOS on Windows and Linux require a working internet connection as it uses the recovery image only, it then downloads the full installer from Apple. The *createistallmedia* script makes an offline installer.

## Post install
- Remove express card icon: Run ```sudo mount -uw / && killall Finder && sudo mv /System/Library/CoreServices/Menu\ Extras/ExpressCard.menu /System/Library/CoreServices/Menu\ Extras/ExpressCard.menu.bak && sudo touch /System/Library/CoreServices/Menu\ Extras/ExpressCard.menu```
- Please re-enable SIP if you don't need it disabled; change ```csr-active-config``` to ```00000000``` reboot and reset nvram
- Check if TRIM is enabled, If not run ```sudo trimforce enable``` to enable it
- Disable ```NVMeFix.kext``` if you don't have an NVMe drive

Finally make sure sleep works properly. You can skip some of these but it will make your machine wake up from time to time. Same as real Macs.
```
sudo pmset standby 0
sudo pmset autopoweroff 0 
sudo pmset proximitywake 0
sudo pmset powernap 0 
sudo pmset tcpkeepalive 0
sudo pmset womp 0
sudo pmset hibernatemode 0
```
The first two need to be ```0``` the rest can be left on if you want.

- Proximity wake can wake your machine when an iDevice is near
- Power Nap will wake up the system from time to time to check mail, make Time Machine backups, etc, etc
- TCP keep alive has resolved periodic wake events after setting up iCloud
- Womp is wake on lan, which is disabled in the BIOS as it (going by other people's experience) might cause issues. I never use WOL, if you do use WOL please try enabling it in the BIOS and leave this setting on, the issues might have been due to bugs that haven been solved by now. Let me know if it works or not.
- Hibernate is sometimes set to 3 in my testing. It could be possible to get hibernation to work by using [HibernationFixup](https://github.com/acidanthera/HibernationFixup) but I haven't tested it. I'm fine with normal sleep.

With hibernation disabled you can delete the sleepimage file and create an empty folder in its place so macOS can't create it again, this saves some space and is optional.
```
sudo rm /var/vm/sleepimage
sudo mkdir /var/vm/sleepimage
```

That's all!

> Tip: Once everything works and you installed and configured all your stuff, create a bootable clone of your system with a trial version of *Carbon Copy Cloner* or *Superduper!*. Don't forget to copy your EFI folder to the clone's EFI partition.

## Big Sur
+ Big Sur needs its own version of Airportitlwm, download the kext [here](https://github.com/zearp/Nucintosh/raw/master/Stuff/AirportItlwm.kext-BigSur.zip) and put it in the kext folder replacing the other one
+ Near the end of the install the system volume will be cryptographically sealed, this will take [some](https://dortania.github.io/OpenCore-Install-Guide/extras/big-sur/#troubleshooting) time
+ To fully disable SIP you need to change ```csr-active-config``` to ```FF0F0000``` in the config
+ ~~In b10 you'll need to set ```SecureBootModel``` to disabled in the config, else it will panic or bootloop~~ Shouldn't be needed anymore.
+ Error 66 when trying to mount / in read/write mode and/or errors about diskXs5s1 when booting, this is due to apfs snapshots;

Boot into recovery and open a terminal then list the snapshots with ```diskutil apfs listSnapshots diskXs5``` and delete them with ```diskutil apfs deleteSnapshot diskXs5 -uuid UUIDHERE```.

Replace diskX with the correct disk, if you only have one disk it will be disk1s5. The UUID is the string above each snapshot.

## Apple/3rd party bluetooth and wifi
For both 1st and 3rd party you will need a [supported](https://dortania.github.io/Wireless-Buyers-Guide/) wifi/bluetooth combo card and an adapter (see below) to convert it to M key. As far as I know compatible M key combo cards don't exist. 

3rd party cards will need these kexts: [AirportBrcmFixup](https://github.com/acidanthera/AirportBrcmFixup) + [BrcmPatchRAM](https://github.com/acidanthera/BrcmPatchRAM), read the instructions on the repo's and you'll be up and running in no time. I've tested the very affordable DW1820A in many machines including the NUC and it works great. For some cards you may need to create an entry under devices in the config that disables ASPM, this only needed if you have issues with sleep.

1st party is my preffered option. Grab an Apple 6+12 pin to m.2 M-key [converter card](https://dortania.github.io/Wireless-Buyers-Guide/Airport.html) and go native with something like the BCM94360CS2. Please note the number of antenna connectors. Some have more than 2, so you'll have to add some antenna's and maybe even mod your case. Though there is some room under the plastic lid, it can fit internal antennas like [this](https://ae01.alicdn.com/kf/HTB1AAiAKVXXXXc4aXXXq6xXFXXX4/2pcs-Internal-Antennas-40cm-15-7-inches-IPEX-MHF4-2-4-5G-wifi-antennas-for-BCM94352Z.jpg). The lid can be removed with some strategic force and there's a hole to route the wires trough too. I would use those and leave the standard antenna's connected to the Intel module. They're very cheap and the antenna connectors on the Intel module are very fragile.

One big plus of going native is that you gain HID-proxy. This means that when there is no OS running the Airport card will proxy any paired HID bluetooth devices to the machine as usb devices. This means you can enter the BIOS or boot menu using the bluetooth keyboard and mouse. This is not a feature you will find on many other cards, including the the one Intel put in here. Even expensive bluetooth cards often can not do this. But Apple has added it even in the cheap BCM943224PCIEBT2 Airport card. I've personally tested that card and it still works fine in Catalina and Big Sur by setting ```Kernel -> Patch -> 0``` to true. Big Sur will also need [a patched](https://github.com/barrykn/big-sur-micropatcher/tree/main/payloads/kexts) IO80211.kext.

Some sellers on AliExpress have converter cards that already have [the small 1.25mm pitch jst](https://github.com/zearp/Nucintosh/blob/master/Stuff/NUC8-m2adapter.jpg?raw=true) connector on it. It connects to one of the two internal usb ports. I use one without issues in my NUC. They usally list them as NUC8 compatible and cost a bit more than other converter cards.

Those other cards (and 3rd party ones) do not come with this connector so you'd have to make your own. My cheaper eBay card came with a cable with standard internal usb header and a cable without any plugs so you can attach your own. Check the listing carefully before ordering. Also make sure it converts to M key and once you have it that the spacing pillar is in the correct position. Don't short the poor Airport out.

- The two internal usb ports are already mapped in the USBPorts.kext, if you made your own map you'll need to make a new map if you use the internal usb headers
- When using a 1st or 3rd party combo card you need to disable both bluetooth and wifi in the BIOS and also remove any Intel related bluetooth and wifi kexts
- You will also need to remove the config block for HS10 from [Info.plist](https://github.com/zearp/Nucintosh/blob/master/EFI/OC/Kexts/USBPorts.kext/Contents/Info.plist#L158-L166) inside USBPorts.kext, without this step bluetooth won't work after sleep. On 1st party cards it gets "stuck" in HID-proxy mode; bluetooth mouse and keyboard may still work but not optimally and laggy.

You'll also want to set your region to ```#a``` as it allows for full 80mhz channel width on ac cards. It might not be 100% legal depending on where you live. I've used this method on a few DW1820A cards and the speed increase was pretty amazing. This method may also apply when using real Apple cards, you will need add [AirportBrcmFixup](https://github.com/acidanthera/AirportBrcmFixup) on 1st party cards. To change the region simply add the following boot flag ```brcmfx-country=#a```.

One last thing to remember is that waking the machine from sleep using bluetooth devices will not work. This is due to power being cut to the module. The module does start itself up very fast. By the time my screen wakes up my bluetooth devices are already reconnected. There is way to bypass this but it includes either modding your adapter card or [making your own](https://github.com/BbIKTOP/M.2-key-M-to-wifi). I've asked some sellers on AliExpress to produce this card but didn't have any luck. If you can make it or know a seller who's willing to make it please let me know.

## ThunderBolt
Should work as long as ThunderBolt security is set to legacy mode. Thanks to [crp724](https://github.com/zearp/Nucintosh/issues/3) for confirming. He also confirmed eGPU works in his Mantiz TB3 enclosure. I assume that if eGPU works then all other ThunderBolt stuff works as well.

## Intel bluetooth and wifi
+ Wifi works and can be managed using native tools, speeds are still slow but connections are stable
+ Bluetooth works for HID devices such as mouse, keyboard and audio stuff but connectiosn are flaky. It may also not wake up from sleep properly

## Natively supported bluetooth dongle
I often use these cheap dongles from [eBay](https://www.ebay.co.uk/itm/1PCS-Mini-USB-Bluetooth-V4-0-3Mbps-20M-Dongle-Dual-Mode-Wireless-Adapter-Device/324106977844) that work in macOS out of the box. When going this route don't forget to disable the Intel bluetooth kexts in the config and also disable bluetooth in the BIOS when using a dongle. You will also need to map the port it connects to as internal else sleep will be dodgy. You can do this easily by setting the port type to 255 in the USBPorts.kext info.plist file. You can find the port identifier (example HS03) with Hackintool.

## Not working/untested
+ Card reader (sort of works with v2.3-beta2 of [this](https://github.com/cholonam/Sinetek-rtsx) kext)
+ IR receiver (it shows up in ioreg but no idea how to make macOS use it like on some MBP)
+ ~~Handoff/AirDrop are not supported (yet) on Intel chips~~ Should work now, untested by me
+ ~~4K [might need](https://github.com/acidanthera/WhateverGreen/blob/master/Manual/FAQ.IntelHD.en.md#lspcon-driver-support-to-enable-displayport-to-hdmi-20-output-on-igpu) some additional parameters and a new portmap~~ 4K confirmed working. Thanks again to [crp724](https://github.com/zearp/Nucintosh/issues/3)
+ You tell me!

## Performance, power and noise
While benchmarks don't really represent real life it can be handy when testing. In my tests undervolting didn't have any impact on Geekbench results scores. But using CPUFriend can have some impact on (immediate) performance depending on which power profile you select.

* Without CPUFriend: ~915 / ~4000
* With CPUFriend: 
  - Performance: same as without
  - Balanced performance: same as without
  - Balanced power savings: ~875 / ~3800
  - Maximum power savings: ~715 / ~3300

The default kexts provided give you the best performance and still lowers the lowest clockspeed to 800mhz. Which lowers heat and power consumption a bit. I didn't notice any difference between the performance and balanced performance profiles but I only ran some quick tests. It is pretty easy to create [your own](https://dortania.github.io/OpenCore-Post-Install/universal/pm.html#using-cpu-friend) profile or disable both CPUFriend kexts in the config.

### Noise
In order to reduce noise I've setup a custom fan profile, disabled the option that the fan can be turned off and set a 25% duty cycle for both CPU and RAM. The idle temps are slightly higher but the noise is a lot less. I've also limited the sustained tdp to 28 watts to match the CPU itself. The peak tdp has been left to its default of 50 watts. With CPUFriend I've set the lowest frequency to 800mhz and a applied a mild undervolt of -50 on the CPU and CPU cache and -25 on the iGPU. A duty cycle of 21 or lower gives me a silent computer but its not ideal to run the fans lower than 25%.

> Note: No longer using a fan, passive cooling ftw!

### Passive cooling
Received my [Akasa](http://www.akasa.com.tw/search.php?seed=A-NUC45-M1B) case. To my surprise it does a better job than the stock cooler. It's not cheap and the case is not finished very smoothly (it can hurt you lol). I have mine verically and didn't use any of the end cheeks, only the feet. It would just introduce more options to hurt myself ;-)

It works really well. So good I have set the power setting in the BIOS to max performance. It idles around 35-40c (with undervolt) which is just fine considering my ambient temperature is around 25c. When put under load it doesn't get anywhere near 80c. I've ran the matrix test from ```stress-ng``` for a while and it stayed [stable around 70c](https://github.com/zearp/Nucintosh/blob/master/Stuff/passive_cooling.png) the whole test.

My only complaint is the rough finish. I wish they would've skipped on those cheeks and spend the money saved on a smooth finish, but thats besides the point of this thing. The silence is worth the occasional scratch.

## Todo
+ Get rid of FakePCIID.kext and do what it does with a config entry or ACPI patch, feel free to submit a PR

## Credits
+ https://github.com/acidanthera
+ https://github.com/OpenIntelWireless
+ https://dortania.github.io/OpenCore-Install-Guide/config.plist/coffee-lake.html
+ https://github.com/Rashed97/Intel-NUC-DSDT-Patch/commit/47476815b52f8e4c97e8f85df158c9ab1b6ecedd
+ https://github.com/csrutil/NUC8I5BEH
+ https://github.com/honglov3/NUC8I7BEH
+ https://github.com/sarkrui/NUC8i7BEH-Hackintosh-Build
