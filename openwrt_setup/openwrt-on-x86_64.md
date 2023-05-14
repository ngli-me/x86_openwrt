# OpenWRT on x86_64
## [Source](https://gist.github.com/pjobson/3584f36dadc8c349fac9abf1db22b5dc)

This is a very brief tutorial on getting OpenWRT installed on a regular computer, it assumes you know your way around Linux.  If you find this and need additional details, please like, subscribe, and comm... oh wait this isn't youtube, just comment.

This is how I got OpenWRT going on a Mini ITX [Intel DH67CF](https://ark.intel.com/content/www/us/en/ark/products/50092/intel-desktop-board-dh67cf.html) with an [Intel G870](https://ark.intel.com/content/www/us/en/ark/products/53493/intel-pentium-processor-g870-3m-cache-3-10-ghz.html) CPU with 4GB of RAM.

## What You'll Need

* 2 USB Sticks
* Linux Live ISO
* Latest OpenWRT combined-ext4 Image

    * As of this writing this version is [18.6.2](https://downloads.openwrt.org/releases/18.06.2/targets/x86/64/)

Use `dd` and create your live Linux ISO.

Format the second USB stick ext4, then `gunzip` the combined-ext4 image, then copy it to the USB stick.

## Installation

Boot off your live Linux USB.

Insert your second USB once linux fully loads.

Use `dd` to write your combined-ext4 image to your hard drive.

Once `dd` completes, open gparted and resize the second partition to around 4GB.

Reboot removing the USB sticks.

## First Boot

Grub should automatically boot to OpenWRT.

You may have to hit enter a couple of times if the boot seems to hang, it'll drop you to the command prompt and complain that there's no password.

Edit your `/etc/config/network` file with `vi`.

You'll want to modify your `lan` interface giving it a static IP within your network.

Here's mine for example:

    config interface 'loopback'
        option ifname 'lo'
        option proto 'static'
        option ipaddr '127.0.0.1'
        option netmask '255.0.0.0'

    config globals 'globals'
        option ula_prefix 'fdb0:9eff:bb07::/48'

    config interface 'lan'
        option type 'bridge'
        option ifname 'eth0'
        option proto 'static'
        option ipaddr '10.10.10.222'
        option netmask '255.255.255.0'
        option gateway '10.10.10.1'

        option dns '10.10.10.1'
        option ip6assign '60'

Then do `service network reload` and you should be able to ping your gateway and outside the network.

You should now be able to get into the gui from any computer in the same subnet.

## Configuration

### Update Packages

`opkg update`

### Packages I Use

`opkg install vim-full nano usbutils pciutils`

### Use `bash`

`opkg install bash`

Edit `/etc/passwd` and change your shell to `/bin/bash`.

#### bash completion

    opkg install ca-bundle ca-certificates libustream-openssl openssl-util
    mkdir ~/bin
    cd ~/bin
    wget https://raw.githubusercontent.com/pjobson/bash-completion/master/bash_completion

    chmod +x ~/bin/bash_completion

    echo ". ~/bin/bash_completion" >> ~/.profile

### Mounting

I like to have a larger `/root` partition and a `swap`.

`opkg install fdisk kmod-fs-ext4 e2fsprogs swap-utils block-mount`

Use `fdisk -l` to list your partitions.

Should show something like this:

    Disk /dev/sda: 93.2 GiB, 100030242816 bytes, 195371568 sectors

    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos

    Disk identifier: 0xc1e0f78c

    Device     Boot    Start       End   Sectors  Size Id Type

    /dev/sda1  *         512     33279     32768   16M 83 Linux
    /dev/sda2          33792   8423423   8389632    4G 83 Linux

From here use `fdisk /dev/sda`.  Create a new primary partition about the size of your RAM for your swap.  Then create a secondary primary partition for your `/root` partition.  Write the changes and do `fdisk -l` again.  Should display something like this:

    Disk /dev/sda: 93.2 GiB, 100030242816 bytes, 195371568 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos
    Disk identifier: 0xc1e0f78c

    Device     Boot    Start       End   Sectors  Size Id Type
    /dev/sda1  *         512     33279     32768   16M 83 Linux
    /dev/sda2          33792   8423423   8389632    4G 83 Linux
    /dev/sda3        8423424  16812031   8388608    4G 83 Linux
    /dev/sda4       16812032 195371567 178559536 85.1G 83 Linux

Format your `sda4` as ext4 with `mkfs.ext4 /dev/sda4`, then make your `sda3` swap space with `mkswap /dev/sda3` then `swapon /dev/sda3`.

View your memory and swap space with `free -k`.

Mount your `sda4` to `/root`, `mount /dev/sda4 /root`.


Enable `fstab` with:

    /etc/init.d/fstab enable

Create your `fstab` with

    block detect > /etc/config/fstab

Edit your `fstab` with `vi`, make it look like this your UUIDs will not be all zeros.

    config 'global'
      option  anon_swap '0'
      option  anon_mount  '0'
      option  auto_swap '1'
      option  auto_mount  '1'
      option  delay_root  '5'
      option  check_fs  '0'

    config 'mount'
      option  target  '/mnt/sda1'
      option  uuid  '00000000-0000-0000-0000-000000000000'
      option  enabled '0'

    config 'mount'
      option  target  '/mnt/sda2'
      option  uuid  '00000000-0000-0000-0000-000000000000'
      option  enabled '0'

    config 'swap'
      option  uuid  '00000000-0000-0000-0000-000000000000'
      option  enabled '1'

    config 'mount'
      option  target  '/root'
      option  uuid  '00000000-0000-0000-0000-000000000000'
      option  enabled '1'

Now reboot and your mount and swap should be good to go.

### Adblocking

`opkg install adblock luci-app-adblock`

Reload luci and you should find Adblock under Services.

### Setting Up Git

`opkg install git git-http ca-bundle libustream-openssl wget`

Generate your ssh keys.

    mkdir -p ~/.ssh

    dropbearkey -t rsa -f ~/.ssh/id_rsa
    dropbearkey -y -f ~/.ssh/id_rsa | sed -n 2p > ~/.ssh/id_rsa.pub

Add your ssh key to github.

    cat ~/.ssh/id_rsa.pub

Add to: [https://github.com/settings/keys](https://github.com/settings/keys)

Git will not work correctly with ssh from the server, this is the workaround.

    mkdir ~/bin
    cd ~/bin
    wget https://raw.githubusercontent.com/pjobson/onion_omega2p_experiments/master/bin/ssh-git
    chmod +x ssh-git

Edit your `.profile` and add:

    export GIT_SSH=~/bin/ssh-git
    export GIT_AUTHOR_NAME="USER NAME"
    export GIT_AUTHOR_EMAIL="user@email.address"
    export GIT_COMMITTER_NAME=$GIT_AUTHOR_NAME

    export GIT_COMMITTER_EMAIL=$GIT_AUTHOR_EMAIL
    export PATH=~/bin:$PATH

Then source your `.profile`.

Now you should be able to connect to github.

