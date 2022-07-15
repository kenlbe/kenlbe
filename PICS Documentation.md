# PICS Configuration Guide
## To write list

3. Config changes
4. DNS Settings
5. Graylog Settings
6. Barracuda setup
7. LDAP
8. Proxy settings
9. IPTables
10. chromium policies


## Clean TMS / Boot Server Setup
Start by installing OS of choice (Ubuntu 22.04 LTS Server Edition is what we used for this guide.)

    sudo apt update && sudo apt upgrade -y
    sudo apt install dnsmasq ncdu nfs-kernel-server -y
    
then add the following to the /etc/dnsmasq.conf file
    
    sudo nano /etc/dnsmasq.conf
	port=0
	enable-tftp
	tftp-root=/data/tftp/boot
	dhcp-range=192.168.30.20,192.168.30.200,2m
	pxe-service=0,"Raspberry Pi Boot"

Following this you need to amend the /etc/exports with any mountable filesystems. These all can exist fine as its the /data/tftp/boot/cmdline.txt that specifies the loaded NFS file system.

    sudo nano /etc/exports
	/data/nfs/<folder to filesystem>
	/data/nfs/master	*(ro,sync,no_subtree_check,no_root_squash)
	/data/nfs/thameside	*(ro,sync,no_subtree_check,no_root_squash)
	/data/nfs/addiewell	*(ro,sync,no_subtree_check,no_root_squash)

Now create the directories for the files, both NFS and TFTP.

    sudo mkdir /data/tftp/boot/
    sudo mkdir /data/nfs/

Change the owner of the /data/tftp/boot folder so it can be accessed by anyone:
    
    sudo chown 777 /data/tftp/boot/

For the boot files you can flash a Pi with PiOS64bit, on the SD card copy the contents of the BOOT folder over to the /data/tftp/boot/ folder. Delete the config.txt and cmdline.txt prior to adding to the server as these choose the file system and parameters to boot.

Now start up the following services.

    sudo systemctl enable dnsmasq
    sudo systemctl enable dnsmasq.service
    sudo systemctl enable rpcbind
    sudo systemctl enable systemd-networkd
    sudo systemctl enable nfs-kernel-server		#If this one fails do: sudo systemctl restart nfs-kernel-server

Amend netplan to set a static IP for the server for convenience sake.... DO NOT tab in this file... its YAML. If you get errors google the syntax. I recommend "tabbing" with 2 spaces.

    sudo nano /etc/netplan/00-install-config.yaml
    network:
        version: 2
    	ethernets:
    	  ens160:
    	  #dhcp: true
    	    addresses:
    		- 192.168.30.5/24
    		routes:
    		- to: default
    		  via: 192.168.30.1
    		nameservers:
    		   search: [thameside.inroom.technology]
    		   addresses: [1.1.1.1,8.8.8.8]
    sudo netplan apply

Give the server a reboot after as a precautionary. 
    
    sudo reboot -h now
    
The above should be done! You just need an nfs to boot and the bootfiles if they differ along with the config.txt and cmdline.txt amendments to boot the relevant NFS.

## Ripping Image Files onto the Boot Server
Plug USB into server with the boot image on from Scoffy.

    lsblk   #find the usb (usually sbd1 or sdf1)
    
Now create a mount directory if not already done and mount the USB to said directory.

    sudo mkdir /mnt/usb
    sudo mount /dev/sdb1 /mnt/usb
    lsblk                           #to check its mounted correctly
    cd /mnt/usb
    ls                              #to check that the files are showing fine

Now add two additional directories so you can mount the image file on the USB.

    sudo mkdir rootmnt/
    sudo mkdir bootmnt/

Then use the kpartx on the image file
    
    kpartx -v a /mnt/usb/<Path-To-Image-File.img>

It will output the paths its then using to be mounted.

    sudo mount /dev/mapper/loop<number from above> rootmnt/
    sudo mount /dev/mapper/loop<number from above> bootmnt/
    
Now change to these directories and find the one with the system files with the ls command.

    cd bootmnt/
    ls

Once you've got the right folder you can move these files like so:
    
    sudo cp -r /mnt/usb/bootmnt/* /data/nfs/<SITE-NAME>/

You'll also need to add the right config.txt and cmdline.txt to the /data/tftp/<sitename> folders for any you want to boot.
Uncomment any you want to boot from /etc/dnsmasq.conf and restart dnsmasq for it to take effect to boot.

## Bootloader
Flash a Pi with clean image of Raspberry Pi OS Lite 64bit.
    
    sudo apt update && sudo apt upgrade -y
    
Browse to the following directory and check the current version of the bootloader

    cd /lib/firmware/raspberrypi/bootloader/stable/
    vcgencmd bootloader_version
    
Use ls or dir command and find this file, then make a copy of the bootloader with:
    
    cp pieeprom-2022-04-26.bin pi4modified290622.bin

Extract the config.txt from this bootloader with:
    
    rpi-eeprom-config pi4modified290622.bin > config.txt
    sudo nano config.txt
    
Repackage this into a third new BIN file with:

    rpi-eeprom-config --out pi4mod290622.bin --config config.txt pi4modified290622.bin

Now you need to generate secure signature for the image...

    curl https://raw.githubusercontent.com/raspberrypi/rpi-eeprom/master/rpi-eeprom-digest > rpi-eeprom-digest
    chmod +x rpi-eeprom-digest
    ./rpi-eeprom-digest -i pi4mod290622.bin -o pieeprom.sig
    
Once you have the modified .bin and the pieeprom.sig use WinSCP (or FTP client) to pull the files. You can then put these files on a fat32 format micro SD card and flash a Pi by inserting the SD card, powering the Pi, wait for the green light to flash slowly, then removing the SD card. The Pi will now be flashed with the custom bootloader.

### Putting a Pi into Read/Write Mode
It is required to enter a Pi into read write mode to make changes to the image, as the Pi will be in read-only mode to ensure things like cache and history are not written to the image.
You will find references to read-only in the below files:

    TMS Server - /etc/exports 
    /data/nfs/master *(ro,sync,no_subtree_check,no_root,squash)
    TMS Server - /data/tftp/boot/cmdline.txt
    ~end of line / elevator=deadline fastboot noswap ro

After amending these files, restart the following services

    sudo systemctl restart rpcbind
    sudo systemctl restart nfs-kernel-server
    sudo systemctl restart dnsmasq

### Screen Resolution Settings
Depending on the style of monitor / TV used on site with the PICS terminals, the settings must be changed to ensure it fits the screen properly.

You will find the screen size settings in the following file:

    TMS Server - /data/tftp/boot/config.txt

    hdmi_group:1=2
    hdmi_mode:1=39
    hdmi_force_hotplug:1=1

The reference numbers pertaining to each resolution / refresh rate can be found on this link https://onlinelibrary.wiley.com/doi/pdf/10.1002/9781119415572.app3 

### Location and Configuration of Chromium Autostart
To get the Pi’s to boot to the correct web page in Chromium, there must be some configuration done to the following file:

    TMS Server - /data/nfs/master/skel*/openbox/autostart
    *please note there may be a difference in the folder names (I have seen skel-xdg, skel1 etc)

    chromium-browser --disable-dev-tool --start-maximized 'http://gbsergatweb01.gatwick.inroom.technology/login/index_<SITENAME>.html'


To work out what link needs to be placed in the above line of the file, refer to End Barracuda session on Browser Window / Tab Close.

### Chromium Policies
The Chromium policies control many aspects of the browser experience. You will find these in the following location:

    TMS Server - /data/nfs/master/etc/chromium-browser/policies/managed/

In this folder are a multitude of .json files which control different policies within Chromium. On configuring Yarl’s Wood, we encountered “Print is blocked” when attempting to Print from Chromium. We found that entering “chrome://print” into the URLWhitelist file allowed this to work.

    {
        "URLWhitelist": ["https://*", "http://*", "chrome://print"]
    }
    
### Update HTML on Intranet Pages
When creating a new PICS site which points back to Gatwick’s Web Server (GBSERGATWEB01), the web pages should be copied into a new directory in the /var/www/html/ directory. For example, the Yarl’s Wood Intranet pages are found in /var/www/html/yw. 

In the new directory with the HTML files, “Gatwick” will need to be changed to your new site name. In order to find where this appears, you can use the following command in the directory given:

    GBSERGATWEB01 -  /var/www/html/NEW_SITE
    grep -r 'Gatwick / SiteNameToReplace' ./

Note – The style of quotation marks you use around ‘Gatwick’ matter, as UNIX only like certain ones.
If you do not want to grep, you can simply enter each HTML file in the directory and use Ctrl+W to find “Gatwick” and replace it manually.

    sudo nano /var/www/html/SITE_NAME/<PAGE>.html
    
### Adding CUPS Printer to Image
To print from the PICS terminals, the printer must be added as an option using cups-client. Follow the instructions below to add the printer

    sudo apt install cups-client
    sudo lpadmin -h 192.168.21.10 -p PRINTER_NAME -m lsb/usr/cupsfilters/Generic-PDF_Printer-PDF.ppd
    lpstat -l -h #Use this to check status of connected printers

### End Barracuda session on Browser Window / Tab Close
Once the browser window is closed on a Terminal, the Barracuda session must log the user out, so the next user cannot access the internet using someone else’s credentials. In order to do this, a few files need to be changed per site.

    GBSERGATWEB01 – /etc/nginx/sites-enabled/default
    
In this file, we can see an entry for location /login shown below. This means that when you go to link http://gbsergatweb01.gatwick.inroom.technology/login it logs you out of the Barracuda sessions.

    location /login {
    alias /var/www/logout/;
    }

A file in the following directory then needs to be created.

    GBSERGATWEB01 – /var/www/logout/
    
In this directory, copy the index.html file and rename it as index_SITENAME.html. This will be the new home page which appears when the browser window is closed. This file will need to be updated with the new home URL on the following line
    
    function loaded() {
                window.location.replace("http://gbsergatweb01.gatwick.inroom.technology/<SITENAME>/");
    };

Lastly, the autostart file needs to be edited to reflect the new page we just created.

    TMS Server – /data/nfs/master/etc/skel*/openbox/autostart

    chromium-browser --user-agent $var --disable-dev-tool --start-maximized 'http://gbsergatweb01.gatwick.inroom.technology/login/index_yw.html'
    
This will reset the login status from the Barracuda once the browser window closes. Set this to /login/index_SITENAME.html, as /login/ is the location specified in the NGINX file we looked at earlier.

### Clear Cache and History of Pi
Clearing the cache and history of the Pi is essential to ensure that there are no cached items on the base image after testing / working in Read-Write mode.

    PICS Terminal – Run the following commands
    sudo rm-rf /etc/skel-pi/.config/chromium
    sudo rm-rf /etc/skel-pi/.cache/chromium

This should be done just before you place the Pi back into Read-only mode using the steps in this document.

### Block Third Party Cookies
This is a policy which blocks third-party cookies.
    
    TMS Server – sudo nano /data/nfs/master/etc/chromium-browser/policies/managed/BlockThirdPartyCookies.json
    {
        "BlockThirdPartyCookies" : true,
        "CookiesBlockedForUrls": ["http://www.google.co.uk", "[*.]google.co.uk",
        "https://consent.youtube.com"],
    }
    
### Running "apt update" on a Terminal
When attempting to run any commands which require an internet process (such as sudo apt upgrade), you may find that the Pi cannot reach the internet. This is caused by the Barracuda which is blocking unauthenticated traffic. See command below how to authenticate when running these commands.
    
    PICS Terminal – sudo 'http_proxy=http://<USERNAME>:<PASSWORD>@192.168.22.10:3128' apt update




 

