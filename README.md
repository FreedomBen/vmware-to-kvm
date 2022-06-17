# Notes on converting a VMware VM to KVM

Much help from:  http://linux-hacking-guide.blogspot.com/2015/05/convert-vmware-virtual-machine-to-kvm.html

## Example Conversion of Metasploitable 2 VM

Download zip file from:  https://sourceforge.net/projects/metasploitable/files/Metasploitable2/

Some documentation about the VM:  https://docs.rapid7.com/metasploit/metasploitable-2/

```bash
cd ~/Downloads

# Unzip the file
unzip metasploitable-linux-2.0.0.zip

cd Metasploitable2-Linux/

# Convert the vmdk file to a qcow2
qemu-img convert -f vmdk Metasploitable2-Linux/Metasploitable.vmdk -O qcow2 Metasploitable.qcow2
```

We're gonna move to a new directory where we can easier permissions

```bash
cd ~/
mkdir vmisos
mv ~/Downloads/Metasploitable2-Linux/Metasploitable.qcow2 vmisos/

# Chown the qcow2 file and vmisos dir so libvirt can read it
sudo chown qemu:qemu -R vmisos
chmod o+x vmisos
```

Download the python script for converting vmx to libvirt's xml style.  Use the python script in the repo at `vmware2libvirt.py` to convert the vmx file to libvirt's xml

```bash
../vmware-to-kvm/vmware2libvirt.py -f ../Metasploitable2-Linux/Metasploitable.vmx > Metasploitable.xml

# The python script buggers the UUID, so we need to fix it
# Remove the `b'` and `'` from the uuid in the xml file
sed -i -e "s|<uuid>b'|<uuid>|g" Metasploitable.xml
sed -i -e "s|'</uuid>|</uuid>|g" Metasploitable.xml
```

On Fedora, need to symlink because `kvm` is not in `/usr/bin/kvm` like `virsh` expects it to be.  It's in `/usr/bin/qemu-kvm`

```bash
ln -s "$(which qemu-kvm)" /usr/bin/kvm

# Example:
ln -s /usr/bin/qemu-kvm /usr/bin/kvm
```

Import the new machine using `virsh`.  Might need to `dnf install libvirt-client` first:

```bash
# Make sure you're in the right hypervisor connection
virsh -c qemu:///session list --all
# or
virsh -c qemu:///system list --all

# Import into the session hypervisor connection (that's where my gnome boxes VMs are)
virsh -c qemu:///system define Metasploitable.xml
```

Before booting the VM, we gotta fix the disk type and location.  See "Original instructions for fixing settings" below for more details

Note:  If you put the qcow2 file in a different directory (like we did above), you will need to modify the location.  You can make the changes manually using `virsh edit Metasploitable2-Linux`

```bash
# Automated
virsh dumpxml Metasploitable2-Linux > ms2.xml
sed -i -e "s|<driver name='qemu' type='raw'|<driver name='qemu' type='qcow2'|g" ms2.xml
sed -i -e "s|<source file='qemu' type='raw'|<driver name='qemu' type='qcow2'|g" ms2.xml
virsh define ms2.xml

# Manual
virsh edit Metasploitable2-Linux
# Change <driver name='qemu' type='raw'/> to <driver name='qemu' type='qcow2'/>
# and <source file='/home/ben/Downloads/Metasploitable2-Linux/Metasploitable.vmdk'/>
# to <source file='/home/ben/Downloads/isos/Metasploitable.qcow2'/>
```

### Fix selinux permissions.

If you can't boot it due to a permission error, you probably need to fix selinux permissions.

Easiest way is to just move the files somewhere already set up and `restorecon` them:

```bash
# List approved places
semanage fcontext -l | grep -i svirt

# mv Metasploitable.qcow2 ~/.local/share/gnome-boxes/
mv Metasploitable.qcow2 ~/.local/share/libvirt/images/

restorecon -R ~/.local/share/libvirt
```

```bash
# Look for the relevant error message
sudo ausearch -m avc

# Export to a file
sudo ausearch -m avc --raw > metasploitable-qcow2-selinux.txt

# Edit the file to remove events that aren't the one we care about
vim metasploitable-qcow2-selinux.txt

# Pipe to audit2allow
cat metasploitable-qcow2-selinux.txt | audit2allow -M metasploitable-qcow2-selinux

# Look at policy for sanity:
vim metasploitable-qcow2-selinux.te

# Apply policy
semodule -i metasploitable-qcow2-selinux.pp

# Or add label
semanage fcontext -a -t svirt_home_t "/home/[^/]+/Downloads/vmisos(/.*)?"
restorecon -R /home/ben/Downloads/vmisos/

# If need to delete the label
semanage fcontext -d -t svirt_home_t "/home/[^/]+/Downloads/vmisos(/.*)?"
```

Profit!


#### Original instructions for fixing settings:

From:  http://linux-hacking-guide.blogspot.com/2015/05/convert-vmware-virtual-machine-to-kvm.html

8) Edit the config file for the VM and modify entry for the virtual disk file and change file type to 'qcow2'

```
[root@meru iso]# virsh edit Metasploitable2-Linux
```

Locate the following lines:

```xml
    <disk type='file' device='disk'>
      <driver name='qemu' type='raw'/>
      <source file='/iso/Metasploitable2-Linux/Metasploitable.vmdk'/>
      <target dev='hda' bus='ide'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
```


And make the following changes:

```xml
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/iso/Metasploitable.qcow2'/>
      <target dev='hda' bus='ide'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
```

Save the changes and exit.
