# The Red Hat Enterprise Linux 7 boot process
Modern computer systems are complex combinations of hardware and software. Starting from an undefined, powered-down state to a running system with a (graphical) login prompt requires a large number of pieces of hardware and software to work together. The following list gives a high-level overview of the tasks involved for a physical x86_64 system booting Red Hat Enterprise Linux 7. The list for x86_64 virtual machines is roughly the same, but some of the hardware-specific steps are handled in software by the hypervisor.

1. The machine is powered on. The system firmware (either modern UEFI or more old-fashioned BIOS) runs a Power On Self Test (POST), and starts to initialize some of the hardware.

   Configured using: The system BIOS/UEFI configuration screens, typically reached by pressing a certain key combination—e.g., F2—early during the boot process.

2. The system firmware searches for a bootable device, either configured in the UEFI boot firmware or by searching for a Master Boot Record (MBR) on all disks, in the order configured in the BIOS.

   Configured using: The system BIOS/UEFI configuration screens, typically reached by pressing a certain key combination—e.g., F2—early during the boot process.

3. The system firmware reads a boot loader from disk, then passes control of the system to the boot loader. On a Red Hat Enterprise Linux 7 system, this will typically be grub2.

   Configured using: grub2-install

4. The boot loader loads its configuration from disk, and presents the user with a menu of possible configurations to boot.

   Configured using: /etc/grub.d/, /etc/default/grub, and (not manually) /boot/grub2/grub.cfg.

5. After the user has made a choice (or an automatic timeout has happened), the boot loader loads the configured kernel and initramfs from disk and places them in memory. An initramfs is a gzip-ed cpio archive containing kernel modules for all hardware necessary at boot, init scripts, and more. On Red Hat Enterprise Linux 7, the initramfs contains an entire usable system by itself.

   Configured using: /etc/dracut.conf

7. The boot loader hands control of the system over to the kernel, passing in any options specified on the kernel command line in the boot loader, and the location of the initramfs in memory.

   Configured using: /etc/grub.d/, /etc/default/grub, and (not manually) /boot/grub2/grub.cfg.

   The kernel initializes all hardware for which it can find a driver in the initramfs, then executes /sbin/init from the initramfs as PID 1. On Red Hat Enterprise Linux 7, the initramfs contains a working copy of systemd as /sbin/init, as well as a udev daemon.

   Configured using: init= command-line parameter.

8. The systemd instance from the initramfs executes all units for the initrd.target target. This includes mounting the actual root file system on /sysroot.

   Configured using: /etc/fstab

9. The kernel root file system is switched (pivoted) from the initramfs root file system to the system root file system that was previously mounted on /sysroot. systemd then re-executes itself using the copy of systemd installed on the system.


10. systemd looks for a default target, either passed in from the kernel command line or configured on the system, then starts (and stops) units to comply with the configuration for that target, solving dependencies between units automatically. In its essence, a systemd target is a set of units that should be activated to reach a desired system state. These targets will typically include at least a text-based login or a graphical login screen being spawned.

   Configured using: /etc/systemd/system/default.target, /etc/systemd/system/
