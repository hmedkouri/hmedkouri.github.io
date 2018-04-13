[Source](https://oliverviebrooks.com/2017/12/05/unlocking-luks-volumes-without-local-access/ "Permalink to Unlocking LUKS Volumes Without Local Access |  Oliver VieBrooks")

# Unlocking LUKS Volumes Without Local Access |  Oliver VieBrooks

[Home][1]

* [Contact][2]
* [Links][3]
* [About][4]

# Unlocking LUKS Volumes Without Local Access

_Tuesday, December 5th, 2017_

Recently, I’ve implemented full-disk encryption on every machine at the lab.

For the 30 or so Linux boxes, this meant setting up [LUKS][5]. We’re a Ubuntu shop here, so that meant following the instructions [here][6]. Actual setup of encryption is outside of the scope of this document. I may write an article on it at some point in the future.

Having set up encryption, when the system boots the user is prompted to enter a password at the console. Given the amount of time I spend working from remote, this is a problem. I can’t reboot any boxes without being on-site, and any wild power events or reboots render all 30 of our servers unusable until they’re manually reset.

What I outline here is the first step in a solution. The second step will come in a later article and explain how to mass-automate the startups. This article covers setting up dropbear so that you can use SSH to log into your system at boot time and unlock the system from remote. There are probably a host of other uses for this hack – remote wiping of the drives is the first thing that springs to mind.

There are a few guides out there for this, but none of them work (because of improper dropbear configuration). That said, I found useful information at the following places:

* [Unlocking a LUKS encrypted root partition remotely via SSH][7]
* [Unlocking Ubuntu Server 16 encrypted LUKS using Dropbear SSH][8]
* [How do I get dropbear to actually work with initramfs?][9][Unable to ssh remote unlock encrypted ubuntu server 15.04 using dropbear/initramfs][10]
* [Remote unlocking LUKS encrypted LVM using Dropbear SSH in Ubuntu Server 14.04.1 (with Static IP)][11]
* [SSH to decrypt encrypted LVM during headless server boot?][12]

Anyway, without further ado, here’s how to set up your Ubuntu Server 17 boxes for remote LUKS unlock. I used nano here, but you can use any text editor of your choice.

**Step 1. Install dropbear and busybox**
    
    
    sudo apt-get install dropbear busybox
    

You will get a warning here as it completes: “dropbear: WARNING: Invalid authorized_keys file, remote unlocking of cryptroot via SSH won’t work!” – ignore it.

**Step 2. Activate BUSYBOX and DROPBEAR in initramfs**
    
    
    sudo nano /etc/initramfs-tools/initramfs.conf
    

Set the “BUSYBOX” option to “y” and add a line below it that says “DROPBEAR=y”

**Step 3. Generate our keys, convert the openssh key to dropbear format, and copy all of the files into /etc/dropbear-initramfs where they belong.**
    
    
    dropbearkey -t dss -f dropbear_dss_host_key
    dropbearkey -t rsa -f dropbear_rsa_host_key
    dropbearkey -t rsa -f id_rsa.dropbear
    /usr/lib/dropbear/dropbearconvert dropbear openssh id_rsa.dropbear id_rsa
    touch id_rsa.pub
    dropbearkey -y -f id_rsa.dropbear |grep "^ssh-rsa " > id_rsa.pub
    touch authorized_keys
    cat id_rsa.pub >> authorized_keys
    sudo cp dropbear_* /etc/dropbear-initramfs/
    sudo cp id_* /etc/dropbear-initramfs/
    sudo cp authorized_keys /etc/dropbear-initramfs/
    

**Note, if you don’t HAVE a /etc/dropbear-initramfs folder, do the following:**
    
    
    sudo mkdir /etc/initramfs-tools/root
    sudo mkdir /etc/initramfs-tools/root/.ssh
    sudo cp dropbear_* /etc/initramfs-tools/root/.ssh/
    sudo cp id_* /etc/initramfs-tools/root/.ssh/
    sudo cp authorized_keys /etc/initramfs-tools/root/.ssh/
    

**Step 4. Set dropbear to start **
    
    
    sudo nano /etc/default/dropbear
    

Change NO_START=1 to NO_START=0

**Step 5. install the crypt_unlock script**
    
    
    sudo nano /etc/initramfs-tools/hooks/crypt_unlock.sh
    sudo chmod +x /etc/initramfs-tools/hooks/crypt_unlock.sh

crypt_unlock.sh:
    
    
    
    #!/bin/sh
    
    PREREQ="dropbear"
    
    prereqs() {
      echo "$PREREQ"
    }
    
    case "$1" in
      prereqs)
        prereqs
        exit 0
      ;;
    esac
    
    . "${CONFDIR}/initramfs.conf"
    . /usr/share/initramfs-tools/hook-functions
    
    if [ "${DROPBEAR}" != "n" ] && [ -r "/etc/crypttab" ] ; then
    cat > "${DESTDIR}/bin/unlock" << EOF
    #!/bin/sh
    if PATH=/lib/unlock:/bin:/sbin /scripts/local-top/cryptroot; then
    kill `ps | grep cryptroot | grep -v "grep" | awk '{print $1}'`
    # following line kill the remote shell right after the passphrase has
    # been entered.
    kill -9 `ps | grep "-sh" | grep -v "grep" | awk '{print $1}'`
    exit 0
    fi
    exit 1
    EOF
      
      chmod 755 "${DESTDIR}/bin/unlock"
      
      mkdir -p "${DESTDIR}/lib/unlock"
    cat > "${DESTDIR}/lib/unlock/plymouth" << EOF
    #!/bin/sh
    [ "$1" == "--ping" ] && exit 1
    /bin/plymouth "$@"
    EOF
      
      chmod 755 "${DESTDIR}/lib/unlock/plymouth"
      
      echo To unlock root-partition run "unlock" >> ${DESTDIR}/etc/motd
      
    fi
    

I found this script [here][13]. I’ve inlined it on this page in case the original goes away.

**Step 6. Update initramfs with the changes**
    
    
    sudo update-initramfs -u
    

**Step 7. Disable dropbear on your booted system.**
    
    
    sudo update-rc.d dropbear disable
    

**Step 8. Download and convert the key for use with putty**  
Download the id_rsa file you created above. You’ll need to convert it from RSA to a putty key.  
Import it in puttygen.exe (Conversions/Import Key)  
Save the private key.

**Step 9. Create a putty session**  
Open putty as you normally would.  
Set Connection/Data/Auto-login username to “root”.  
On the Connection/SSH/Auth screen, browse and load the private key you saved, using the “Browse…” button on the “Authentication parameters” panel.  
Save this session.

**How to unlock your system (finally)**  
Finally, you should be able to connect to your rebooted (and awaiting unlock) server. Open putty, load your session, and connect.  
If you’ve done everything properly, you’ll get a prompt.  
At the prompt, use the command “unlock”.  
If you’ve entered the proper password, your server will reboot and dropbear will disconnect. Wait a bit for it to complete booting and you should be able to log in as normal.

* * *

© OliverVieBrooks.com 2017

[1]: https://oliverviebrooks.com
[2]: https://oliverviebrooks.com/contact/
[3]: https://oliverviebrooks.com/links/
[4]: https://oliverviebrooks.com/about/
[5]: https://guardianproject.info/code/luks/
[6]: https://help.ubuntu.com/community/ManualFullSystemEncryption
[7]: http://blog.neutrino.es/2011/unlocking-a-luks-encrypted-root-partition-remotely-via-ssh/
[8]: http://blog.netpacket.co.uk/2016/12/05/unlocking-ubuntu-server-16-encrypted-luks-using-dropbear-ssh/
[9]: https://askubuntu.com/questions/640815/how-do-i-get-dropbear-to-actually-work-with-initramfs
[10]: https://askubuntu.com/questions/620136/unable-to-ssh-remote-unlock-encrypted-ubuntu-server-15-04-using-dropbear-initram
[11]: https://stinkyparkia.wordpress.com/2014/10/14/remote-unlocking-luks-encrypted-lvm-using-dropbear-ssh-in-ubuntu-server-14-04-1-with-static-ipst/
[12]: https://unix.stackexchange.com/questions/5017/ssh-to-decrypt-encrypted-lvm-during-headless-server-boot
[13]: https://gist.github.com/gusennan/712d6e81f5cf9489bd9f
