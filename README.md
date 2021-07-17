# inkPalm-5-EPD105-root

**Follow anything in this guide at your own risk. I will not accept any responsibility if you are left with a bricked device, and/or a device that is damaged or impaired in some way.** These steps are a reconstruction of what I was doing - I haven't tested the resulting files linked to in this guide.

The inkPalm 5 is a 5.2" eReader with an Allwinner 32-bit ARM B300 CPU and 1GB RAM running Android 8.1 sold in China under Xiaomi's Moaan brand.

[![inkPalm 5 with a terminal open showing its rooty goodness](https://i.ibb.co/JrGxN0C/1.png)](https://i.ibb.co/vLTsbgD/1.png)

* [inkPalm-5-EPD105-root](#inkpalm-5-epd105-root)
* [Background](#background)
  * [I considered rooting impossible because](#i-considered-rooting-impossible-because)
  * [And there was light](#and-there-was-light)
* [Why not root](#why-not-root)
* [Fine, just show me the rooting steps, dude](#fine-just-show-me-the-rooting-steps-dude)
  * [Prepatched image](#prepatched-image)
  * [OR patching your own image](#or-patching-your-own-image)
    * [Flashing the patched kernel image](#flashing-the-patched-kernel-image)
      * [Using recovery to do it](#using-recovery-to-do-it)
      * [The easy but untested way: using fastboot](#the-easy-but-untested-way-using-fastboot)
* [microG](#microg)
  * [Setting up signature spoofing](#setting-up-signature-spoofing)
  * [Installing microG](#installing-microg)
* [How do I unroot?!](#how-do-i-unroot)
* [What about OTAs?](#what-about-otas)
* [Launcher plug](#launcher-plug)
* [TODO](#todo)

# Background

After two weeks of trying, I am proud to announce that I, qwerty12, have finally achieved root on the inkPalm 5. The bootloader is not locked but the main hurdle was actually getting the device's boot.img to feed to Magisk.

## I considered rooting impossible because

* There is no firmware download available for this device - no boot.img
* The one OTA that has been issued does not contain a full boot.img, but simply a delta update to the existing boot partition.
* There was no obvious backdoor in a system application that I could see
* The inkPalm's recovery was locked down: no running adbd and `adb sideload` would only accept signed zips
* I found no secret `fastboot` command to dump partitions
* Being an Allwinner device, you can get to [FEL](https://linux-sunxi.org/FEL) mode but not [FES](https://linux-sunxi.org/FES) mode, which lets you dump partitions, due to not having fes1.fex. A dead end.
* The existing exploits I found for Android 8.1 relied on knowing kernel-specific addresses, which you can't get with a kernel to actually look at
  * Unsurprisingly, the kernel source is nowhere to be found, so I couldn't build my own using the existing config.gz

## And there was light

On a hunch, I looked again at the OTA, particularly focusing on its certificate. Googling the first line of the pubkey in the otacert file brought results - the ZIP was signed with the test key included with the Android recovery source! I could now sign my own ZIPs that `adb sideload` would accept.

I could not get the recovery to execute `busybox` from the ZIP file. Maybe user error or SELinux troubles, but I ended up switching tack: I mounted the system partition and executed its `toybox` binary (one isn't present on the recovery partition) to `dd` the kernel block partition into a folder readable by booted Android proper.  SELinux prevented copying to /data, /private, /vendor. /cache worked but I couldn't access anything from there when booted in Android. My last resort worked: /system.  
Back in Android, copying that file to /sdcard and having Magisk patch it worked. I made another ZIP and had the recovery flash the Magisk-patched image successfully.

# Why not root

* This has only been tested on one device, mine...
  * My inkPalm is kind of a little messed up from my experiments of using the MiReader's fes1.fex and u-boot.fex. Case in point: `fastboot boot` with the Magisk-patched image caused my IP to hang. Eventually, I got the inkPalm to boot the original kernel; however, unplugging the USB cable caused it to power off. I can't remember how I fixed it exactly, but it was a combination of reflashing the original kernel in recovery, applying just the fex files from the OTA update and booting the u-boot.fex from the OTA for good measure.
  * I caused the OEM unlock option to show in developer options (a `setprop` command of some sort) and I ticked it. Not sure it was needed, or what it changes; the bootloader should already be unlocked to the best of my knowledge. The setting did persist after wiping /data, though. I'm too scared to turn it off.

* **There is no original firmware available to flash**. The implications of this one should be obvious. If they're not, then don't follow this guide.

* The stock recovery, for which there is no alternative, is pretty restricted. You might be able to get out of a bootloop by erasing /data at the cost of losing your data. There is the untested possibility of maybe writing a file to disable Magisk through a sideloaded ZIP.

* There's [this comment](https://old.reddit.com/r/eink/comments/n1yn02/new_xiaomi_pocket_reader_hands_on/gwiv097/) regarding downsides of rooting the MiReader. (Note: After a month of having a rooted inkPalm, I have had no stability or boot issues. I never, and do not plan to, install(ed) Xposed on the inkPalm. Just sayin'.)

* That application you bought from the Play Store that you want to run on the inkPalm may work without Google Play Services

# Fine, just show me the rooting steps, dude

This assumes you have ADB set up. This is something that has been covered in [countless](https://github.com/philips/inkpalm-5-adb-english) [guides](https://github.com/epodegrid/epd106-ADB) on the Internet. It is not something I will provide help for.

## Prepatched image

Preferred, because you don't have to touch the /system partition on your inkPalm yourself (which may break OTAs), but it requires trusting these images aren't malicious, and you must be running the `MAS_EPD105_L8AM105_V05_210518` build (first OTA offered on 2021-05-21). You can check what build you're currently running by opening the settings in the Moan/original launcher and tapping the icon with the upwards facing arrow in a circle.

1. In an `adb shell` run `getprop ro.fota.version`. If the result is anything other than `MAS_EPD105_L8AM105_V05_210518`, then do not continue. Move onto [patching your own image](#or-patching-your-own-image) or see if doing a software update gets you this build.

2. Download [MAS_EPD105_L8AM105_V05_210518_magisk_v23_dmv_not_preserved_signed.zip](https://github.com/qwerty12/inkPalm-5-EPD105-root/releases/tag/MAS_EPD105_L8AM105_V05_210518)

3. Reboot the inkPalm into recovery mode: `reboot recovery`

4. Use the touch screen and/or volume + power buttons to choose the "Apply update from ADB" option

5. On the computer, run `adb sideload path\to\MAS_EPD105_L8AM105_V05_210518_magisk_v23_dmv_not_preserved_signed.zip` (making the obvious substitution)

6. When it's done, cross your fingers and choose the first option in the recovery (reboot system now) to boot back into Android

7. If all went well, connect to Wi-Fi, tap the Magisk icon and do what it says.

8. Oh, do yourself a favour and run `su` in an `adb shell` at least once. You'll be glad you did if your inkPalm 5 is bootlooping.

9. That's it! Move onto the [microG](#microg) section if that interests you.

## OR patching your own image

This involves writing a file to /system, which may prevent you from installing future OTAs.

1. Download [dump_kernel_to_system_signed.zip](https://github.com/qwerty12/inkPalm-5-EPD105-root/raw/main/dump_kernel_to_system_signed.zip)

2. Reboot the inkPalm into recovery mode: `reboot recovery`

3. Use the touch screen and/or volume + power buttons to choose the "Apply update from ADB" option

4. On the computer, run `adb sideload path\to\dump_kernel_to_system_signed.zip` (making the obvious substitution)

5. When it's done, choose the first option in the recovery (reboot system now) to boot back into Android

6. Back in an `adb shell`, run `cp /system/bimg.img /sdcard/`

7. Install the [Magisk APK](https://github.com/topjohnwu/Magisk/releases/latest)

8. Open Magisk, tap install

9. I don't think the AVB/dm-verity option needs to be checked, so I left it unchecked. YMMV. The inkPalm doesn't encrypt itself even if you enable a PIN.

10. Choose bimg.img from the storage and let Magisk do its thing

11. Transfer the magisk_patched*.img file in your Downloads folder onto the PC

### Flashing the patched kernel image

#### Using recovery to do it

1. Rename your magisk_patched.img file to boot.img

2. Clone this repository or [download a ZIP](https://github.com/qwerty12/inkPalm-5-EPD105-root/archive/refs/heads/main.zip)

3. Using your favourite archive manager, add your boot.img to the root folder of kernel_flashing_template.zip

4. In a command prompt, run `java -jar signapk-1.0.jar -w testkey.x509.pem testkey.pk8 kernel_flashing_template.zip kernel_flashing_template_signed.zip`

    (If you run into problems, make sure you have JRE 8 installed.)

5. Follow the instructions in [Prepatched image](#prepatched-image) from step three onwards, substituting the zip file mentioned there with kernel_flashing_template_signed.zip

#### The easy but untested way: using fastboot

<details>
  <summary>Again, I must state that <b>I have not tested this</b> way, but it is much, much simpler if it works.
</summary>
  
  1. Run `adb reboot bootloader`

  2. Run `fastboot flash \path\to\magisk_patched*.img`

  3. If the flash is successful, run `fastboot reboot`

  4. Set up Magisk as per [Prepatched image](#prepatched-image)

</details>

# microG

I would recommend doing this only if you're trying to run applications that require Google Play Services (GPS for short). One person who installed microG on his MiReader ended up removing it because supposedly there was increased battery drain. 

[Aurora Store](https://f-droid.org/en/packages/com.aurora.store/) is an excellent Google Play Store alternative client that runs on unrooted devices without any form of Google Play Services installed. Although I have seen quite a few warnings from various people since saying not to sign in because Google may terminate your account for going against their terms of service. I haven't read of any confirmed cases though. If you don't want to take the risk, just stick to using it with the anonymous accounts it offers. 

## Setting up signature spoofing

In order for microG to pretend it is the real GPS, [signature](https://github.com/microg/GmsCore/wiki/Signature-Spoofing) [spoofing](https://github.com/FriendlyNeighborhoodShane/MinMicroG/blob/master/install.md#microg-signature-spoofing) must be set up. As those two links show, there's many ways of achieving it.

I used [fOmey's Smali Patcher](https://forum.xda-developers.com/t/module-smali-patcher-7-3.3680053/) with these options:

* mock locations
* mock providers
* signature spoofing

Just as long as signature spoofing is ticked, you can choose whatever you want (although I haven't personally tested anything other than what I listed).  
Enabling the signature *verification* patch is not needed. You can tick it if you want, but you should be aware that the Play Store will update itself to an unpatched version without support for IAPs through microG if you do so.

Just be aware of having the module enabled if performing an OTA system update. You might end up in a bootloop if you do not disable the module beforehand and regenerate it afterwards.

Once you have the Magisk module it generated for your inkPalm installed and enabled, you can move onto the next step:

## Installing microG

1. With your Wi-Fi enabled, open Magisk.
2. Install the "Busybox for Android NDK" module and reboot
3. Grab your [preferred variant](https://github.com/FriendlyNeighborhoodShane/MinMicroG/blob/master/README.md) of the MinMicroG installer from [here](https://github.com/friendlyneighborhoodshane/minmicrog_releases/releases/latest)
4. Install the ZIP with Magisk and reboot
5. You should see new a microG Settings icon. Go crazy.

I went for the `MinimalIAP` version. 

If you don't want the Play Store to be enabled all the time for battery reasons, [FreezeYou!](https://android.izzysoft.de/repo/apk/cf.playhi.freezeyou) makes disabling it a doddle.

# Bromite WebView

The included WebView is very old, from Android 8.1. You will not recieve updates for it from the Play Store. It can however be replaced with [Bromite's](https://www.bromite.org/) SystemWebView.

1. Download [BromiteWebView.zip](https://github.com/qwerty12/inkPalm-5-EPD105-root/raw/main/BromiteWebView.zip)

2. Download the latest [SystemWebView](https://www.bromite.org/system_web_view) for ARM (not ARM64)

3. Rename the downloaded file (which is currently `arm_SystemWebView.apk`) to `webview.apk` (this is case-sensitive)

4. Using your favourite archive manager, open BromiteWebView.zip and add webview.apk to the system/app/webview folder inside the ZIP

5. Open the downloaded webview.apk with your archiver, go to lib/armeabi-v7a and extract the two `.so` files

6. Place the extracted so files into the system/app/webview/lib/arm folder inside the ZIP

7. Transfer the zip to your inkPalm and use Magisk to install it. Reboot.

For updates to the Bromite WebView, either add the [F-Droid](https://www.bromite.org/fdroid) repository or repeat the above process when neeeded.

# How do I unroot?!

I literally have no idea. Remember, the original firmware for this device isn't available.

If you never wrote a single file to /system, then flashing the stock boot image *may* undo most of it, thanks to Magisk's systemless nature. Not having tried that, I cannot say if it is safe to do.

# What about OTAs?

They may not work after this. In my case, I have no plans on installing any more.

From the old version of Android as standard, to an exposed `setprop` method added to the framework that can be called by any unprivileged application, to the OTA ZIPs that are signed with the test recovery key, I do not see what exactly the OTAs could improve on.

Plus, future updates may fix what is shown here. :)

# Launcher plug

If you're in need of a launcher to replace the default, then I recommend [my fork](https://github.com/qwerty12/E-Ink-Launcher/releases/latest) (which tries to work around inkPalm-specific quirks) of [shunf4's fork](https://github.com/shunf4/E-Ink-Launcher) of [Modificator's E-Ink-Launcher](https://github.com/Modificator/E-Ink-Launcher).

# TODO

* A patched stock recovery (TWRP seems unlikely) with root adbd and SELinux disabled. Oh, and include a shell!

* See if it's possible to change the shutdown image
