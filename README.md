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

```
