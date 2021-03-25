# misc

# Fedora core-os full installation

**FEDORA COREOS PODMAN CONTAINER IMAGE**
https://docs.rs/crate/coreos-installer/0.1.2

### make a directory for all fikes `mkdir coreos-files` `cd coreos-files` and use podman to download the coreos iso

```
podman run --privileged --pull=always --rm -v .:/data -w /data \
    quay.io/coreos/coreos-installer:release download -s stable -p metal -f iso
```

### make a fcc file `touch fcct-v0.1.fcc`, and write this down (*replace the ssh-rsa with your key*)

```
variant: fcos
version: 1.3.0
passwd:
  users:
    - name: core
      ssh_authorized_keys:
        - ssh-rsa AAAAB3NzaC...
```

### then, transpile the fcc in a ign (basically it does yaml > json)

```
podman run -i --rm quay.io/coreos/fcct:release --pretty --strict < fcct-v0.1.fcc > transpiled_config.ign
```

### you can validate your ignition file as below. it only checks for bad indentation

```
podman run --pull=always --rm -i quay.io/coreos/ignition-validate:release - < transpiled_config.ign
```

### if the above command doesn't return errors, it's time to burn the iso to a usb stick
write `lsblk` and you will get an output like:

```
sdc      8:16   1  14.9G  0 disk 
├─sdc1   8:17   1   1.6G  0 part /media/username/usb volume name
└─sdc2   8:18   1   2.4M  0 part 
```

you have to see where the USB is mounted, in this case it's `/dev/sdc`
so the command I will write to burn the iso to USB is:

```
sudo dd bs=4M if=/var/home/bogdanbujor/fcos-fcct-ign/fedora-coreos-33.20210217.3.0-live.x86_64.iso of=/dev/sdc conv=fdatasync  status=progress
```

you will have to modify `of=/dev/sdc` with `of=/dev/sdX`, where X can be other letters as well
you will also have to modify the path of the ISO file. in my case it was `if=/var/home/bogdanbujor/fcos-fcct-ign/fedora-coreos-33.20210217.3.0-live.x86_64.iso`. yours could be `if=/var/home/$USER/coreos-files/fedora-coreos...iso`. verify with `pwd` in the folder where you downloaded the ISO and append with the ISO name

### after the `dd` command finishes (which will take some time), you will have to boot from the USB thumb drive on the PC you want fedora installed

then, at the command line, you have to

```
sudo coreos-installer install /dev/sda --insecure-ignition \
    --ignition-url http://10.1.1.188/transpiled_config.ign
```
This

```--ignition-url http://10.1.1.188/transpiled_config.ign```

will change in your case. put your transpiled ignition file on an accesible web server.

### Read more below

https://docs.fedoraproject.org/en-US/fedora-coreos/bare-metal/
https://coreos.github.io/coreos-installer/customizing-install/
https://github.com/coreos/ignition#config-validation
https://docs.fedoraproject.org/en-US/fedora-coreos/authentication/


## Problems with /dev/sda in use?
**basically I wanted to install coreos on top of of fedora**
but `/dev/sda` was in use, and coreos-installer wouldn't launch
so a solution I found was to erase all data from `/dev/sda`

1. `lsblk`
2. `sudo dd if=/dev/zero of=/dev/sda bs=512 count=1`
3. `sudo blkdiscard /dev/sda`
   After blkdiscard (which does a TRIM on the SSD), I gave the PC around 1min rest for the blkdiscard command to propagate (some people say it's necessary, others say it's bs)
4. `systemctl reboot`
