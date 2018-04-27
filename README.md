# Headless AMD APU / iGPU 3D Acceleration on Ubuntu

This guide describes how to get an integrated AMD APU + GPU (or potentially any other iGPU) to accelerate through SSH in headless mode (without a display connected to the computer) on Linux. Steps 1-2 are only necessary if this is your first time setting up the server. Afterwards, only step 3 and step 4 are necessary to enable 3D acceleration. These instructions will be specific for the AMD + Radeon Driver but you may add instructions based on the APU brand + driver you're using. I have not been able to get AMDGPU-Pro to work on Kabini so this guide is aimed at RadeonDriver. This is a work in progress.

### Step 1: INITIAL SETUP OF XSERVER CONFIG:
If we are using the “radeon" driver, we will create an Xorg conf at `/etc/X11/xorg.conf.d/20-radeon.conf`.

In that file we place some basic instructions for X server using our driver:

    Section "Device"
        Identifier "Radeon"
        Driver "radeon"
    EndSection


### Step 2: KERNEL BOOT PARAMETERS (Radeon only)

Edit our kernel bootloader config:
	
    sudo nano /etc/default/grub

In that file add the following parameters. The dpm parameter turns off the gpu driver's power management, which to my knowledge might be one of the culprits which stop the GPU from running all the time but I haven’t tested this and cannot confirm. I just know that this is part of how I got to my solution. You only need the si parameters if you have a sea islands or southern islands APU. The si support parameters enable sea islands and southern islands GCN support for the radeon driver. On newer linux kernels / and ubuntu versions these parameters are included by default but I have included them here just in case a user is on an older Linux kernel and they can’t hurt anything to include anyways if you have one of these cards.

	GRUB_CMDLINE_LINUX_DEFAULT="quiet radeon.dpm=0 radeon.si_support=1 radeon.cik_support=1"

Then update grub in order to write out the config:

	sudo update-grub



### Step 3: START A NEW DISPLAY WITH XORG:
This display will be set to :11 in the following snippet. Note that the only reason we have an & at the end of this command is in order to make this command a background process and be able to continue executing commands in the same terminal window.

	Xorg -noreset +extension GLX +extension RANDR +extension RENDER -logfile ./10.log -config ./xorg.conf :11 &

### Step 4: EXPORT THAT DISPLAY:
Now we export that display in order to enable it. You must export this display in every single shell window/tab in order to have access to the videocard. However you only need to do step 3 in one window.

	export DISPLAY=":11"

### UTILITIES TO TEST & DEBUG 3D ACCELERATION IN SSH

##### GLXGEARS:
GLXGears might complain about vsync. We open it with the vblank_mode parameter. 

	vblank_mode=0 glxgears

##### GLXINFO:
In glxinfo you should have "direct rendering: yes" and see your videocard listed as the "OpenGL renderer." Pipe grep to glxinfo in order to filter relevant data.

	glxinfo | grep ‘render'


##### RADEONTOP:
Radeontop should show videocard in use while GLXGears is running, with moving bars and percentages. Pass the -c parameter to have nice colors.

	sudo radeontop -c


### glmark2:

glmark2 provides benchmarking. You can watch radeontop at the same time to see the load on the GPU in real time.

### ADDITIONAL COMMANDS USEFUL FOR DIAGNOSTICS:

The following command displays the available kernel parameters for “radeon” driver. Very useful for finding a kernel parameter that might enable your particular GPU or unlock something you need.

	modinfo -p radeon

The following command displays kernel drivers available and / or in use:

	lspci -k | grep -iEA5 'vga|3d|display'

The following command displays bootloader output:

	dmesg | egrep 'drm|radeon'

The following command displays loaded drivers:

	lsmod | grep radeon 

Additionally, in `/var/log/` there are numerous Xorg log files which are useful for diagnostics.


### Sources: 
Xdummy, instructions for faking a display: 
http://xpra.org/trac/wiki/Xdummy

Radeon driver xorg configuration: https://jlk.fjfi.cvut.cz/arch/manpages/man/radeon.4

Radeon linux kernel parameters: https://www.x.org/wiki/RadeonFeature/#index4h2

Very useful wiki on configuring radeon driver: https://wiki.archlinux.org/index.php/ATI#Xorg_configuration