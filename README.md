# :sos: Fedora 43 post installation guide for Nvidia users
fedora is honestly very barebones after a clean installation, this distro doesn't even have proper video codecs!, installing and maintaining it its fairly easy, but the post-installation is honestly at the level of being a ritual, so hopefully this guide will streamline it a bit for you!


## :monocle_face: I just installed fedora, now what?
on the welcome page, be sure to enable the **third party repositories**, it will enable some but not all of them
then we proceed to activate some more! the reason is that without this, the flatpak selection on fedora is honestly very limited.
open the konsole and paste:

`sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm`

then we update what we just installed

`sudo dnf group upgrade core`

`sudo dnf4 group install core`

`sudo dnf check-update`


and now we **update our system**

`sudo dnf -y update`

after the update is installed, reboot your system


now we **update our firmware**

this refreshes the firmware database

`sudo fwupdmgr refresh --force`

this shows us what can be updated

`sudo fwupdmgr get-devices`

this fetches the list of the updates

`sudo fwupdmgr get-updates`

and lastly this updates

`sudo fwupdmgr update`


## :dizzy: Improve your connection (maybe):
now, I really recommend going to the system settings and in the connections section, disable ipv6, the reason is that your connection might be very slow with that enabled, you can re-renable it back at any point if you feel uncomfortable with it disabled so don't worry.

I also recommend to go to your firewall settings and set the zone to home instead of workstation, might improve your connection speed too.


## :tophat: Now we add the flathub repository

fedora doesn't include all of the repositories by default, this command enables the flathub flatpaks:

`flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo`


## :watch: Now we set up snapper for BTRFS snapshots and grub support, so we can rollback changes on launch

first, this will make sure the grub menu is always visible

`sudo grub2-editenv - unset menu_auto_hide`

then we add snapper, btrfs assistant and the tools we need

`sudo dnf install snapper libdnf5-plugin-actions btrfs-assistant inotify-tools git make`

now we make a file named snapper.actions, just copy everything in this box and paste it in the terminal

```
sudo bash -c "cat > /etc/dnf/libdnf5-plugins/actions.d/snapper.actions" <<'EOF'
# Get snapshot description
pre_transaction::::/usr/bin/sh -c echo\ "tmp.cmd=$(ps\ -o\ command\ --no-headers\ -p\ '${pid}')"

# Creates pre snapshot before the transaction and stores the snapshot number in the "tmp.snapper_pre_number"  variable.
pre_transaction::::/usr/bin/sh -c echo\ "tmp.snapper_pre_number=$(snapper\ create\ -t\ pre\ -c\ number\ -p\ -d\ '${tmp.cmd}')"

# If the variable "tmp.snapper_pre_number" exists, it creates post snapshot after the transaction and removes the variable "tmp.snapper_pre_number".
post_transaction::::/usr/bin/sh -c [\ -n\ "${tmp.snapper_pre_number}"\ ]\ &&\ snapper\ create\ -t\ post\ --pre-number\ "${tmp.snapper_pre_number}"\ -c\ number\ -d\ "${tmp.cmd}"\ ;\ echo\ tmp.snapper_pre_number\ ;\ echo\ tmp.cmd
EOF
```
now we make snapper configurations for the root subvolume

`sudo snapper -c root create-config /`

verify it worked if you want

`sudo snapper list-configs`

restore the selinux context for the directory

`sudo restorecon -RFv /.snapshots`

verify it if you want

`ls -1dZ /.snapshots`

enable user access to snapper and sync ACL's

`sudo snapper -c root set-config ALLOW_USERS=$USER SYNC_ACL=yes`

this prevents updatedb from indexing the .snaptshots directories, so if you have many it won't slowdown anything

`echo 'PRUNENAMES = ".snapshots"' | sudo tee -a /etc/updatedb.conf`

with this you can see a list of your snapshots, but probably it will tell you that you have no snapshots right now

`snapper ls`

### Now we install grub-btrfs so our snapshots are detected in the grub menu (the menu you have on launch, before fedora boots up)

first we clone the repository

`git clone https://github.com/Antynea/grub-btrfs`

go to its path

`cd grub-btrfs`

these are some fedora-specific changes we have to do, just copy everything and paste it in the terminal, be sure you're in the grub-btrfs path
```
sed -i.bkp \
  -e '/^#GRUB_BTRFS_SNAPSHOT_KERNEL_PARAMETERS=/a \
GRUB_BTRFS_SNAPSHOT_KERNEL_PARAMETERS="rd.live.overlay.overlayfs=1"' \
  -e '/^#GRUB_BTRFS_GRUB_DIRNAME=/a \
GRUB_BTRFS_GRUB_DIRNAME="/boot/grub2"' \
  -e '/^#GRUB_BTRFS_MKCONFIG=/a \
GRUB_BTRFS_MKCONFIG=/usr/bin/grub2-mkconfig' \
  -e '/^#GRUB_BTRFS_SCRIPT_CHECK=/a \
GRUB_BTRFS_SCRIPT_CHECK=grub2-script-check' \
  config
```
now we install it, don't worry if you see a no snapshots found message, its normal

`sudo make install`

and now we enable the service

`sudo systemctl enable --now grub-btrfsd.service`

go back to root and remove the cloned repository

`cd ..`

`rm -rfv grub-btrfs`

congratulations, snapper is fully functional now!

go to btrfs-assistant and create your first snapshot if you wish, enable timeline snapshots if you want them in the snapper settings tab


I recommend to reboot now, then lets go back again to fedora


## :skull: NVIDIA Drivers

the annoying part, even more than setting up snapper

first lets check if you have **secureboot** enabled:

`mokutil --sb-state`

if you do, reboot, access your bios and disable it, its the **easier way to do this**

the hard way, you'll probably have to find a guide about how to set up your secure boot so it works with nivida drivers, its honestly a lot of annoying steps so do yourself a favor and disable it if you don't really care about it

now next step, we install the kernel headers and dev tools:

`sudo dnf install -y kernel-devel kernel-headers gcc make dkms acpid libglvnd-glx libglvnd-opengl libglvnd-devel pkgconfig`

install the nvidia driver

`sudo dnf install -y akmod-nvidia xorg-x11-drv-nvidia-cuda`

now we wait 5 to 15 minutes for the kernel module to be built

you can check if its built with this command:

`modinfo -F version nvidia`

if it gives an error its not built, if you get a number, then it is built

in any case, still give yourself some time and take around 10 minutes to check

once its built, reboot your system.


now check if it worked:

`nvidia-smi`

if you don't see your gpu info, probably something went wrong, try to rebuild your kernels and reboot if it's borked

`sudo akmods --force --kernels $(uname -r)`

then same deal, wait for around 10 minutes, check if the kernel is built and reboot.


### everything went well, but now I see an error while booting fedora!

its probably secureboot disabling the nvidia driver, I told you earlier to disable it!


**if everything is fine**, lets keep going

### if you use a laptop like I do, we might also need to enable the modeset:

`sudo grubby --update-kernel=ALL --args="nvidia-drm.modeset=1"`

and even then, fedora might just outright ignore your nvidia gpu, so for those cases I recommend to do this:

`cd /etc`

`sudo nano environment`

now we add this line:

`DRI_PRIME=1`

now we press ctlr+O and then enter to save the changes and then ctlr+X to close the dialog, after that:

`cd ..`

to go back to root

this will force fedora to use our nvidia gpu as the main gpu, ignoring the integrated one for the most part.

**now reboot**


## :dvd: CODECS

lets make music and videos work

this replaces the dumbed down ffmpeg we have for the real one

`sudo dnf swap -y ffmpeg-free ffmpeg --allowerasing`

this installs the gstreamer plugins:
`
sudo dnf install -y gstreamer1-plugins-{bad-\*,good-\*,base} \
  gstreamer1-plugin-openh264 gstreamer1-libav lame\* \
  --exclude=gstreamer1-plugins-bad-free-devel
  `
and last but not least, install the multimedia groups:

`sudo dnf group install -y multimedia`

`sudo dnf group install -y sound-and-video`

now we install the hardware acceleration, so videos use the gpu

`sudo dnf install -y ffmpeg-libs libva libva-utils`

`sudo dnf install -y libva-nvidia-driver`

as usual, now reboot


now we install microsoft fonts, don't blame me, we still need them.

lets install the dependencies first

`sudo dnf install -y curl cabextract xorg-x11-font-utils fontconfig`

then the fonts

`sudo rpm -i --nodigest --nosignature https://downloads.sourceforge.net/project/mscorefonts2/rpms/msttcore-fonts-installer-2.6-1.noarch.rpm`

and then we update the font cache

`sudo fc-cache -fv`


**if you plan on using appimages** (they're essentially self contained programs), you need to install fuse

`sudo dnf install -y fuse fuse-libs`

then go to the discover app to install gearlever so you can manage them


speaking of...

## :gift: Let's bloat our system!

here's a list of absolutely essential apps you should install on fedora, you can find them in discover, if possible install them from flathub unless I say the opposite:

```
bleachbit (ccleaner for linux basically)

flatseal (manage flatpak permissions)

pika backup (make backups from your home folder and data)

VLC (video player)

gnome disk manager (you'll find it under the name disks)

peazip (to open rar and 7zip files)

fedora media writer (better balena etcher)

```

now if you plan on emulating windows programs, also install:

```
wine (from fedora linux, its the v10 which is pretty recent)

bottles (makes self contained windows installations)

lutris (similar to bottles but aimed at games)
```

as optional flatpaks to install, I personally recommend:

```
gimp (image editor)

gapless (music player)

qimgv (image viewer)

qbitorrent (to download torrents)

steam (download the fedora linux version, the flathub one has been very buggy for me)

vesktop (discord works really poorly on linux, this makes it run really well)

onlyoffice (I find it better than libreoffice)

mission center (to monitor your gpu and stuff)
```

## :shower: now after all that, lets clean up:

clean the package cache:

`sudo dnf clean all`

and remove the orphaned packages you might have:

`sudo dnf autoremove -y`


now you can go to the system settings, default applications and change some like vlc to be your default video player

once you're done, **make a snapshot**, named it something like base and then reboot your system


### :star2: congratulations! you got fedora up and ready for daily use!

feel free to install more flatpaks you might need now, download themes, extra libraries you might need, etc.

at the very least, the minimum install that will make everything else work is there.


I hope the guide wasn't too difficult to follow!

special thanks to everybody who made other guides and videos for me to follow and the fedora community, you all made my system work properly and this is just a compilation of what worked for me.

and of course disclaimer:

### this is not an official guide or anything like that, and you're doing all of this under your own risk
