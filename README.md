# Proxmox-HP-Proliant-Microserver-Gen8-PCIE-Passthrough
-------------------------------------------------------

Wer die Pakete selbst kompilieren möchte kann sich hieran orientieren:

Wie folgt bin ich vorgegangen:

####Abhängigkeiten bzw. benötigte Pakete herunterladen:

```
$ apt-get install git screen fakeroot build-essential devscripts libncurses5 libncurses5-dev libssl-dev bc flex bison libelf-dev libaudit-dev libgtk2.0-dev libperl-dev libperl-dev asciidoc xmlto gnupg gnupg2
```
####Download der Sources:
```
git clone git://git.proxmox.com/git/pve-kernel.git
```
(es sind ca. 4,5 GB)

danach muss der Patch angewendet werden:

Der Patch von Cainsaw sah wie folg aus:

####Patch:
```
--- ubuntu-xenial/drivers/iommu/intel-iommu.c.orig      2016-11-24 08:16:14.101871771 +0100
+++ ubuntu-xenial/drivers/iommu/intel-iommu.c   2016-11-24 08:17:36.581195062 +0100
@@ -4797,11 +4797,6 @@ static int intel_iommu_attach_device(str
        int addr_width;
        u8 bus, devfn;
-       if (device_is_rmrr_locked(dev)) {
-               dev_warn(dev, "Device is ineligible for IOMMU domain attach due to platform RMRR requirement.  Contact your platform vendor.\n");
-               return -EPERM;
-       }
-
        /* normally dev is not mapped */
        if (unlikely(domain_context_mapped(dev))) {
                struct dmar_domain *old_domain;
```

abgewandelt auf den ubuntu-xenial kernel sieht das ganze dann so aus:

```
--- ubuntu-xenial/drivers/iommu/intel-iommu.c   2016-12-05 10:04:59.000000000 +0100
+++ ubuntu-xenial/drivers/iommu/intel-iommu.c.BACKUP    2016-12-19 11:17:35.897891365 +0100
@@ -4807,11 +4807,6 @@
        int addr_width;
        u8 bus, devfn;
-       if (device_is_rmrr_locked(dev)) {
-               dev_warn(dev, "Device is ineligible for IOMMU domain attach due to platform RMRR requirement.  Contact your platform vendor.\n");
-               return -EPERM;
-       }
-
        /* normally dev is not mapped */
        if (unlikely(domain_context_mapped(dev))) {
                struct dmar_domain *old_domain;
```

Um dieses Patchfile zu erstellen muss man wie folgt vorgehen:

```
$ apt-get install patch
```

####Patchfile erstellen:

ubuntu-xenial.tgz entpacken:

```
tar -xvzf ubuntu-xenial.tgz
```

zu ubuntu-xenial/drivers/iommu/intel-iommu.c navigieren:

```
cd ubuntu-xenial/drivers/iommu/
```
```
$ cp ubuntu-xenial/drivers/iommu/intel-iommu.c ubuntu-xenial/drivers/iommu/intel-iommu.c.BACKUP
$ nano ubuntu-xenial/drivers/iommu/intel-iommu.c.BACKUP
```

diese Zeilen entfernen:

```
if (device_is_rmrr_locked(dev)) {
              dev_warn(dev, "Device is ineligible for IOMMU domain attach due to platform RMRR requirement.  Contact your platform vendor.\n");
               return -EPERM;
       }
```

danach wieder in den Hauptordner pve-kernel zurücknavigieren und den patch erstellen:

```
$ diff -u ubuntu-xenial/drivers/iommu/intel-iommu.c ubuntu-xenial/drivers/iommu/intel-iommu.c.BACKUP > remove_mbrr_check.patch
```

darauf sollte das file: remove_mbrr_check.patch
mit etwa folgendem Inhalt im pve-kernel Ordner erscheinen:

```
--- ubuntu-xenial/drivers/iommu/intel-iommu.c   2016-12-05 10:04:59.000000000 +0100
+++ ubuntu-xenial/drivers/iommu/intel-iommu.c.BACKUP    2016-12-19 11:17:35.897891365 +0100
@@ -4807,11 +4807,6 @@
        int addr_width;
        u8 bus, devfn;

-       if (device_is_rmrr_locked(dev)) {
-               dev_warn(dev, "Device is ineligible for IOMMU domain attach due to platform RMRR requirement.  Contact your platform vendor.\n");
-               return -EPERM;
-       }
-
        /* normally dev is not mapped */
        if (unlikely(domain_context_mapped(dev))) {
                struct dmar_domain *old_domain;
```

####Patch integrieren:

Jetzt muss der Patch noch in das Makefile des Kernels gepackt werden, dass die Änderungen beim kompilieren angewendet werden:

```
$ nano Makefile
```
nach der Zeile:

```
cd ${KERNEL_SRC}; patch -p1 <../bridge-patch.diff
```
einfach folgende Zeile einfügen:

```
cd ${KERNEL_SRC}; patch -p1 <../remove_mbrr_check.patch
```
dann speichern und den Kompilierprozess starten:

####Kompilieren:

```
$ make
```
Der Vorgang kann einige Zeit bis zu mehreren Stunden, je nach Hardware, in Anspruch nehmen,..

wenn der Vorgang abgeschlossen ist solltet ihr in eurem Kernel Ordner mehrere .deb Dateien haben.
die werden jetzt installiert:

```
linux-tools-4.4_4.4.35-76_amd64.deb
proxmox-ve_4.4-76_all.deb
pve-firmware_1.1-10_all.deb
pve-headers-4.4.35-1-pve_4.4.35-76_amd64.deb
pve-headers_4.4-76_all.deb
pve-kernel-4.4.35-1-pve_4.4.35-76_amd64.deb
```
####Installation:

installiert werden die via:

```
$ dpkg -i *.deb
```
danach vorsichtshalber noch ein:

```
$ update-initramfs -u
$ update-grub
```
####Einstellungen in Proxmox:

/etc/default/grub anpassen
--->
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"
```
ich brauche nicht mehr in meiner Zeile. Es wird soweit alles gut erkannt


/etc/modules folgendes einfügen um die vfio Module beim boot zu laden:
--->
```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```
Treibermodul auf die Blacklist:
/etc/modprobe.d/blacklist.conf und darin:
```
blacklist hpsa
```
da über lspci -vvv: Kernel driver in use: hpsa angeigt wurde.

danach noch initramfs neu schreiben:
```
$ update-initramfs -u
```

und Grub updaten:
```
$ update-grub
```

danach alles neu starten !!!!!

nach dem Reboot folgendes in die vm.conf eintragen:
--->

```
hostpci0: 07:00.0
```
Mehr braucht es bei mir nicht.

und schon hatte ich den RaidController in meiner Openmediavault VM. Zugriff auf den Controller, auf die Platten und und und..
