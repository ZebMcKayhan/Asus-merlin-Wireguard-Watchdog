# Asus-merlin-Wireguard-Watchdog
Example Wireguard watchdog script for Asuswrt-merlin

Please use this script as you wish, fork it, modify it for your needs, but please share your result.

It have basic checks so selected Wireguard interface is active or enabled. 

Wireguard should normally not require any restarting since there is no connection in that sense. Nothing really changes when you restart it (or should not). If you do find that your Wireguard infact stops working at times and restarting it makes it work it may mean there is an issue lurking in your system. While this watchdog script can benefit you by relieve you of restarting it, you are advised to hunt down the reason for it to stop work. Use the scripts log-files to help you figure out what is happening.

# Install instructions
to download the script:
```sh
curl --retry 3 "https://raw.githubusercontent.com/ZebMcKayhan/Asus-merlin-Wireguard-Watchdog/main/wgc-watchdog" -o "/jffs/scripts/wgc-watchdog" && chmod 755 "/jffs/scripts/wgc-watchdog"
```
For further instructions on how to use the script or how to make it periodically, check under "Using the script"


# About the script
The scripts makes the following checks when executed:
- Wireguard handshake timer existence and within limits
- It pings 8.8.8.8 (or custom ip) using selected interface and checks for response.

If any checks fails it will notify in router syslog and attempt to restart the interface.

A currently commented feature is if it fails it starts a timer that reboots the router if it still fails after 1 hour. If you want to use this feature you have to remove the brackets near the end. I may enable this feature if the script gets mature enough.

To help debugging, the script saves log files of WG output, Routing rules and routes (both main and policy table) as well as firewall Filter, Nat and mangle. Log files are created both before and after the restart so you may do a compare-by-content on these files and figure out what have changed.

The log files are placed in ram folder (/tmp/) so you don't need to worry about wearing down router flash if you end up with an interface restarting every 10min for a longer time.

# Using the script
the script could be executed from ssh by adding the wgc number after it. 1 for wgc1, 2 for wgc2 et.c
```sh
/jffs/scripts/wgc-watchdog 2
```
for a single check on wgc2, by pinging 8.8.8.8. typical for an internet vpn client.

if you want to use this on I.e site-2-site script, 8.8.8.8 may not be available over the tunnel. in such case you may specify the IP it should ping by i.e:
```sh
/jffs/scripts/wgc-watchdog 2 192.168.2.1
```
but beware, the ip has no checks so if it's in wrong format the script may fail or think vpn is not working and try to restart it

The script will output each successful or failed test directly on screen. 
I encourage everyone to test the script this way first. 
Check router syslog for any script outputs, but if all checks pass it will not output anything to syslog.

If the tests fails and the interface is restarted, the script produces log files here:
```sh
/tmp/wgc-watcdog_wgc1_before.log
```
for system state before the interface was restarted. And here
```sh
/tmp/wgc-watcdog_wgc1_after.log 
```
for system state after the interface was restarted.
Comparing these files to each other and you could figure out what is changing.
Please note that the log files contains sensitive info, like your public ip and wg public keys. Do not post them publically without obfuscating them.

The idea is that the script should be run periodically on the router, using cron, like:
```sh
cru a WatchWgc2 "*/10 * * * * /jffs/scripts/wgc-watchdog 2"
```
or if you want to specify custom ip to ping:
```sh
cru a WatchWgc2 "*/10 * * * * /jffs/scripts/wgc-watchdog 2 10.6.0.1"
```

It would be needed to setup this in firmware hook script wgclient-start and removed in wgclient-stop to make this periodic checks start when the client is started and stopped when the client is disabled in the GUI

to do this you will need to set "Enable JFFS custom scripts and configs" to yes in the router gui (Administration -> system) if you have not already done so.

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
