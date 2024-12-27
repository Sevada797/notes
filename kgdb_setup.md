Full guide to set up kernel debugging in virtual box,

Considering you already have you virtual-machine set up

1. Download linux kernel source code , install dependencies needed with `sudo apt-get install build-essential`

2. Enter root directory of kernel source code e.g. `cd linux-6.6.62` then run `make menuconfig` or manually edit the .config 
file and make sure these parameteres are like below

```
CONFIG_GDB_SCRIPTS=y
CONFIG_KGDB=y
CONFIG_KGDB_SERIAL_CONSOLE=y
CONFIG_DEBUG_KERNEL=y
CONFIG_DEBUG_INFO=y
CONFIG_DEBUG_INFO_DWARF4=y #just in case
CONFIG_MAGIC_SYSRQ # This is very very useful
```

3. Run `make` this will take some time...
after run 
`make modules_install`  this will install the neccessary kernel modules in /lib/modules/linux-<verison> folder
then
`make install`  this will copy the neccesary files in /boot folder , to be able to boot into newly built kernel

4. After edit the grub config by running
`sudo nano /etc/default/grub`
add lines like this below, just make sure to replace Linux 6.6.62 with the appropriate name, the same name you see during 
boot if you enter "Advanced options for Ubuntu", additionally you can not change GRUB_DEFAULT and choose whenever you may
want to boot to the kernel with kgdb symbols setup at grub boot menu.

```
GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 6.6.62"
GRUB_CMDLINE_LINUX_DEFAULT="kgdboc=ttyS0,115200 kgdbwait nokaslr"
```

nokaslr is useful for not randomizing kernel memory addresses each time you boot. Because with same memory
addresses between boot sessions, running add-symbols-file for each .ko file is becoming more quick and easy, I will explain 
this in further instructions.

5. Run `sudo update-grub` to apply the changes. with scp or other means transfer the vmlinux file to host machine.
It's at linux kernel source code root directory.

6. Now power off the mahcine, then open the "vbox settings"-->"serial ports" check the "Enable Serial Port" option,
set "Port Mode"-->"Host pipe", and in the input box "Path/Address" fill some non-existing filename like 
e.g. /tmp/vm_serial_pipe , press OK to apply settings, 

7. turn on the virutal-machine, now if you boot to the newly installed kernel, you should see something like
```Waiting for GDB connections...```

If you see this, from host machine run:	`gdb /path/to/vmlinux`  , `target remote /tmp/vm_serial_pipe`
and for now simply type `continue` and hit enter

From virtual-machine:
go to the ~/your_kernel_modeules_sourcedir , `make clean && make`, scp these *.ko files to host machine, preferably at same folder as where vmlinux 
was copied, then reboot again

8. From host machine: quit gdb, rerun `gdb vmlinux` command then after you 
`target remote /tmp/vm_serial_pipe`,
add breakpoint at insmod by this command below
`break do_init_module` and after `continue`

9. Now load modules, (each insmod will call the breaks set earlier)
now go to host machine, and see that breakpoint was hit, run `p mod->name` then `p mod->init` this second command will show
you the memory address at which the modules are being loaded, 
now if `p mod->name` shows you for example your.ko and second command smth like 0xfff...
all is correct, now run `add-symbols-file your.ko` or if your.ko is in other directory `add-symbols-file /path/to/your.ko`
then run `continue` you will hit the breakpoint again if you are running the script reload-yourko, otherwise manually insmod
the other modules, and do exact same steps for them.
now after doing this for 3rd module you can delete the breakpoint, the setup is all done, if at some point there is kernel 
panic, you can debug them with commands like `bt`

Useful tip: create ~/break file, and paste inside
```echo g > /proc/sysrq-trigger```
and each time you need to pause the kernel and get gdb prompt in your host machine, you can do it so by running 
`sudo bash ~/break`

Notes: Make sure that in both machines ~/your_kernel_modeules_sourcedir folder is existent and that they are exact same code
like with `git log` check if both have the same commit hash, so the codes in both folders are identical, 

