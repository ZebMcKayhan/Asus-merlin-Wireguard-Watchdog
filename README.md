# Asus-merlin-Wireguard-Watchdog
Example Wireguard watchdog script for Asuswrt-merlin

This script is still under development so please use it with care. it still have no input checks or even checks if selected Wireguard interface is active or enabled. 

# Install instructions
to download the script:
```sh
curl --retry 3 "https://raw.githubusercontent.com/ZebMcKayhan/Asus-merlin-Wireguard-Watchdog/main/wgc-watchdog" -o "/jffs/scripts/wgc-watchdog" && chmod 755 "/jffs/scripts/wgc-watchdog"
```

# About the script
The scripts makes the following checks when executed:
- Firewall rules related to the interface (filter, nat, mangle)
- Policy route table existence and routes
- Wireguard handshake timer existence and within limits
- It pings 8.8.8.8 using selected interface and checks for response.

If any checks fails it will notify in router syslog and attempt to restart the interface.

A currently commented feature is if it fails it starts a timer that reboots the router if it still fails after 1 hour. If you want to use this feature you have to remove the brackets near the end. I may enable this feature if the script gets mature enough.

# Using the script
the script could be executed from ssh by adding the wgc number after it. 1 for wgc1, 2 for wgc2 et.c
```sh
/jffs/scripts/wgc-watchdog 2
```
for a single check on wgc2

I will output each successful or failed test directly on screen. 
I encourage everyone to test the script this way first. 
Check router syslog for any script outputs, but if all checks pass it will not output anything to syslog.

the idea is that the script should be run periodically on the router, using cron, like:
```sh
cru a WatchWgc2 "*/10 * * * * /jffs/scripts/wgc-watchdog 2"
```

It would be needed to setup this in firmware hook script wgclient-start and removed in wgclient-stop

to do this you will need to enable "custom scripts" in the router gui if you have not already done so.

then edit/create the file that firmware executes when starting a WG client:
```sh
nano /jffs/scripts/wgclient-start
```
populate the file with:
```sh
#!/bin/sh 

if [ "$1" -eq "2" ]; then # only for wgc2
   cru a WatchWgc2 "*/10 * * * * /jffs/scripts/wgc-watchdog 2"
fi
```
adjust according to your needs.

save & exit nano (ctrl+x - y - enter)

then edit/create the file that firmware executes when stopping a WG client:
```sh
nano /jffs/scripts/wgclient-stop
```
and populate with:
```sh
#!/bin/sh 

if [ "$1" -eq "2" ]; then # only for wgc2
   cru d WatchWgc2
fi
```
again, adjust according to your setup.

save & exit nano

now we need to make these files we have created executable:
```sh
chmod +x /jffs/scripts/wgclient-start
```
and
```sh
chmod +x /jffs/scripts/wgclient-stop
```
and that's it!

you will need to restart your WG clients for these files to execute and start the cron job that makes periodically checks.
