## Validity Sensor `138a:0090` and `138a:0097` libfprint driver
#### A linux driver for 2016 ThinkPad's fingerprint readers

[![See it in action!](https://img.youtube.com/vi/dYe8eKaoUSE/0.jpg)](https://www.youtube.com/watch?v=dYe8eKaoUSE)`

Thanks to the amazing work that [nmikhailov](https://github.com/nmikhailov) did in his [prototype](https://github.com/nmikhailov/Validity90/) and [uunicorn](https://github.com/uunicorn/) in [python-validity](https://github.com/uunicorn/python-validity) and [synaWudfBioUsb-sandbox](https://github.com/uunicorn/synaWudfBioUsb-sandbox), I spent some time in getting a libfprint driver for the `138a:0090` (and `138a:0097`) device up...

 * It only works if the device has been initialized using [validity-sensors-tools/](https://snapcraft.io/validity-sensors-tools/)
   - Alernatively, but it's less secure, you can use a Windows installation with VirtualBox (sharing USB) guest or with a Windows installation in bare metal
 * This version works only with fprintd2, if you still use fprintd see the [old version](https://github.com/3v1n0/libfprint/tree/vfs0090-libfprint1).
 * Most of the device interaction and crypto code is coming from the prototype, so basically it needs lots of cleanup, but I hope to rebase this on the code from python-validity
 * Here enroll, verification, led and all the operations work
 * It uses libfprint image comparison algorithm, we might move to in-device check later.

You can test it using the [examples](tree/vfs0090/examples) or just using `fprintd-*` tools (GNOME supports it natively from control center).


### Device initialization and pairing

I recommend using the [validity-sensors-tools/](https://snapcraft.io/validity-sensors-tools/), you can install it in any distro as snap, or you can use it manually from sources located in [my python validity fork](https://github.com/3v1n0/python-validity)

```bash
sudo snap install validity-sensors-tools

# Give it access to the usb devices
sudo snap connect validity-sensors-tools:raw-usb
sudo snap connect validity-sensors-tools:hardware-observe

# Initialize the device
sudo validity-sensors-tools.initializer

# Test the device
sudo validity-sensors-tools.led-test

# This is needed and only works in 138a:0097:
sudo validity-sensors-tools.enroll --finger-id [0-9]

# See other available tools
validity-sensors-tools --help
```

[![Get it from the Snap Store](https://snapcraft.io/static/images/badges/en/snap-store-black.svg)](https://snapcraft.io/validity-sensors-tools)

#### Match on Chip

This is the only supported way by `138a:0097`, so once you've enrolled your fingers with `validity-sensors-tools.enroll` you will be able to re-enroll your fingers in fprintd to use them in linux as well.

This unfortunately can't be done in `138a:0090`, but you can still use a Windows installation (even in VirtualBox) to enroll the prints to save them in the chip and enable the match-on-sensor, this can make the verification faster, safer and higher quality.<br />
Unfortunately there's currently no easy way to implement this in this driver without reverse-engineer the fingerprint template creation that the windows drivers does in host.

#### Ubuntu installation

If you're using ubuntu just use [this PPA](https://launchpad.net/~3v1n0/+archive/ubuntu/libfprint-vfs0090) to get the libfprint TOD packages with vfs0090 sensor support.

Also, in the ubuntu (and derivates) the code that you will use will be based on the [tod submodule](https://gitlab.freedesktop.org/3v1n0/libfprint-tod-vfs0090).

You can enroll your fingers by using the `fprintd-enroll` utility or from UI using `unity-control-center user-accounts` in unity or `gnome-control-center user-accounts` in GNOME (it's the same as going in System settings -> User accounts pane and enable the fingerprint login).

So, in steps (for ubuntu) it would be:
```bash
# Initialize the device
sudo snap install validity-sensors-tools
sudo snap connect validity-sensors-tools:raw-usb
sudo snap connect validity-sensors-tools:hardware-observe
sudo validity-sensors-tools.initializer

# Add the repository and install the tod package (supports both chips)
sudo add-apt-repository -u ppa:3v1n0/libfprint-vfs0090
sudo apt install libfprint-2-tod-vfs0090
```

Then go in system settings (account) and enable the fingerprint login

#### Arch linux Installation

Install packages:
 * `fprintd`
 * `libfprint-vfs0090-git` from AUR

#### Fedora (tested on 28)
- `sudo dnf install -y libusb*-devel libtool nss nss-devel gtk3-devel glib2-devel openssl openssl-devel libXv-devel gcc-c++`
- `git clone https://github.com/3v1n0/libfprint`
- `meson libfprint libfprint/_build && sudo ninja -C libfprint/_build install`

#### NixOS

NixOS has the tod module in nixpkgs-unstable (merged June 2021).
On release 21.05 or newer, and assuming `pkgsUnstable` is an unstable nixpkgs, configure like this:

```
{
  services.fprintd.enable = true;
  services.fprintd.tod.enable = true;
  services.fprintd.tod.driver = pkgsUnstable.libfprint-2-tod1-vfs0090;
}
```

#### Other distros
 - `git clone https://github.com/3v1n0/libfprint`
 - `meson libfprint libfprint/_build && sudo ninja -C libfprint/_build install`


#### fprintd enrolling
```bash
for finger in {left,right}-{thumb,{index,middle,ring,little}-finger}; do fprintd-enroll -f "$finger" "$USER"; done
```

#### Help testing (only for `138a:0090`)

It would be nice if you could help in tuning the value of the `bz3_threshold`, as that's the value that defines how different should be the prints, and so it's important for having better security. I've set it to `12` currently, but of course increasing the number of prints we enroll or the image quality that could be increased.

Using `fprint_demo` or monitor fprintd from journalctl you should be able to see the values such as `fpi_img_detect_minutiae` and `fpi_img_compare_print_data` in the log, like

```
fp:debug [fpi_img_new] length=82944
fp:debug [fpi_imgdev_image_captured]
fp:debug [fpi_img_detect_minutiae] minutiae scan completed in 0,080257 secs
fp:debug [fpi_img_detect_minutiae] detected 18 minutiae
fp:debug [print_data_new] driver=15 devtype=0000
fp:debug [fpi_img_compare_print_data] score 9
fp:debug [fpi_img_compare_print_data] score 12
fp:debug [fpi_img_compare_print_data] score 18
fp:debug [fpi_img_compare_print_data] score 10
fp:debug [fpi_img_compare_print_data] score 12
```

The score is the value the print got for you, compared to each sample that fprint saves... And to match it needs to reach the said threshold (so 12 for now). For my fingers this value seems secure enough, but.... Let's see if we can increase it.
