## Validity Sensor `138a:0090` libfprint driver
#### A linux driver for 2016 ThinkPad's fingerprint readers

[![IMAGE ALT TEXT HERE](https://img.youtube.com/vi/dYe8eKaoUSE/0.jpg)](https://www.youtube.com/watch?v=dYe8eKaoUSE)`

Thanks to the amazing work that [nmikhailov](https://github.com/nmikhailov) did in his [prototype](https://github.com/nmikhailov/Validity90/), I spent some time in getting a libfprint driver for the `138a:0090` device up...

 * It only works if the device has been initialized with a Windows VirtualBox (sharing USB) guest or with a Windows installation in bare metal
 * Most of the device interaction and crypto code is coming from the prototype, so basically it needs lots of cleanup, but I noticed Nikita is already on that, so I'll be happy to integrate it in next iterations (the thing that actually took the most was having proper fprintd state machines).
 * Here enroll, verification, led and all the operations work
 * First initialization is the most problematic thing so far, we're still looking on it.
 * It uses libfprint image comparison algorithm, we might move to in-device check later.

You can test it using `fprint-demo` available in various distro's repositories, or just using `fprintd-*` tools (GNOME supports it natively from control center).


#### Ubuntu installation

If you're using ubuntu just use [this PPA](https://launchpad.net/~3v1n0/+archive/ubuntu/libfprint-vfs0090) to get the libfprint packages with vfs0090 sensor support.


Once you've added the ppa you can test it with the `fprint_demo` application (`fprint-demo` package) or use it for your desktop by installing the `libpam-fprintd`package.

You can enroll your fingers by using the `fprintd-enroll` utility or from UI using `unity-control-center user-accounts` in unity or `gnome-control-center user-accounts` in GNOME (it's the same as going in System settings -> User accounts pane and enable the fingerprint login).

So, in steps (for ubuntu) it would be:
 - `sudo add-apt-repository -u ppa:3v1n0/libfprint-vfs0090`
 - `sudo apt install libpam-fprintd`
 - Go in system settings (account) and enable the fingerprint login

#### Arch linux Installation

Install packages:
 * `fprintd`
 * `libfprint-vfs0090-git` from AUR

#### Fedora (tested on 28)
- sudo dnf install -y libusb*-devel nss nss-devel gtk3-devel glib2-devel openssl openssl-devel libXv-devel gcc-c++
- `git clone https://github.com/3v1n0/libfprint`
- `cd fprint && ./autogen.sh && make && sudo make install`

#### Other distros
 - `git clone https://github.com/3v1n0/libfprint`
 - `cd fprint && ./autogen.sh && make && sudo make install`


#### fprintd enrolling
```bash
for finger in {left,right}-{thumb,{index,middle,ring,little}-finger}; do fprintd-enroll -f "$finger" "$USER"; done
```

#### Help testing

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
