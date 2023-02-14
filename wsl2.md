* remember to run `wsl --update`

* I'll be using WSL2 because I like making things difficult.
* This is split into two different sections, steps 1-8 are initial configuration steps and then step 9-10 will be for on-demand lab spin up.
* with reference to: https://github.com/ocroz/wsl2-boot or https://stevegy.medium.com/wsl-2-static-ip-341603d84401


* CURRENTLY there's a problem where each VM is assigned the same IP address... the last configured with `ip`.  I'm going to look at https://github.com/wikiped/WSL-IpHandler to see if this will help, it is called out in the purpose statement: "All running WSL Instances have the same random IP address within WSL SubNet although default SubNet prefix length for some reason is 16 (which is enough for 65538 ip addresses!)."

* build for WSL-IpHandler, this seems like it might work for just IP addressing the WSL2 VMs...
* it doesn't: https://github.com/wikiped/Wsl-IpHandler/issues/36
  * but maybe this can help https://superuser.com/a/1715457/91174
```
Invoke-WebRequest https://raw.githubusercontent.com/wikiped/Wsl-IpHandler/master/Install-WslIpHandlerFromGithub.ps1 | Select -ExpandProperty Content | Invoke-Expression
Import-Module Wsl-IpHandler
man install-wsliphandler
Install-WslIpHandler -WslInstanceName ubuntu_control -GatewayIpAddress 192.168.143.1 -WslInstanceIpAddress 192.168.143.111 -UseScheduledTaskOnUserLogOn
Install-WslIpHandler -WslInstanceName ubuntu_workernode1 -GatewayIpAddress 192.168.143.1 -WslInstanceIpAddress 192.168.143.112 -UseScheduledTaskOnUserLogOn
```

* Here's another idea:
  * to create a bridge, you must launch `wsl` as an admin.
```
#!/usr/bin/env bash
instance_num=$1
#if [ -e /run/netns/]

# Create the bridge that will be common to all instances.
# Only a `wsl --shutdown` will terminate the bridge, unless
# otherwise manually removed.
if [ ! -e /sys/devices/virtual/net/br1 ]
then
    ip link add name br1 type bridge
    ip addr add 10.0.0.253/24 brd + dev br1
    ip link set br1 up
fi

# Add namespace for this instance
if [ ! -e /run/netns/vnet${instance_num} ]
then
    ip netns add vnet${instance_num}
fi

# Adds a veth pair.  The vethX
# side will reside # inside the namespace 
# and be the primary NIC inside that namespace.
# The br-vethX  end will reside in the primary
# namespace.
ip link add veth${instance_num} type veth peer name br-veth${instance_num}
ip link set veth${instance_num} netns vnet${instance_num}
# Give it a unique IP based on the instance number
ip netns exec vnet${instance_num} \
    ip addr add 10.0.0.${instance_num}/24 dev veth${instance_num}
ip link set br-veth${instance_num} up
# Add the bridged end of the veth pair
# to br1
ip link set br-veth${instance_num} master br1
ip netns exec vnet${instance_num} \
    ip link set veth${instance_num} up

# Set the default route in the namespace
ip netns exec vnet${instance_num} \
    ip route add default via 10.0.0.253
# Enable loopback fort he namespace
ip netns exec vnet${instance_num} \
    ip link set up dev lo
# Set up NAT for return traffic
iptables \
    -t nat \
    -A POSTROUTING \
    -s 10.0.0.0/24 \
    -j MASQUERADE
# Enable forwarding
sysctl -w net.ipv4.ip_forward=1

# Optional - Start a namespace for the 
# default WSL user (UID 1000).
# You can exit this namespace normally
# via the `exit` comamnd or Ctrl+D.
default_username=$(getent passwd 1000 | cut -d: -f1)
nsenter -n/var/run/netns/vnet${instance_num} su - $default_username
```




1. build a reasonable host compute network on your host

```
# use this: https://github.com/skorhone/wsl2-custom-network
cd $repo\
git clone https://github.com/skorhone/wsl2-custom-network.git
cd wsl2-custom-network
import-module -Name .\hcn

#mind the indents when dealing with the here string
#https://learn.microsoft.com/en-us/windows-server/networking/technologies/hcn/hcn-json-document-schemas
#the only type supported in Windows 10 is ICS
$network = @"
{
        "Name" : "WSL",
        "Flags": 9,
        "Type": "ICS",
        "IPv6": false,
        "IsolateSwitch": true,
        "MaxConcurrentEndpoints": 1,
        "Subnets" : [
            {
                "ID" : "FC437E99-2063-4433-A1FA-F4D17BD55C92",
                "ObjectType": 5,
                "AddressPrefix" : "192.168.143.0/24",
                "GatewayAddress" : "192.168.143.1",
                "IpSubnets" : [
                    {
                        "ID" : "4D120505-4143-4CB2-8C53-DC0F70049696",
                        "Flags": 3,
                        "IpAddressPrefix": "192.168.143.0/24",
                        "ObjectType": 6
                    }
                ]
            }
        ],
        "MacPools":  [
            {
                "EndMacAddress":  "00-15-5D-52-CF-FF",
                "StartMacAddress":  "00-15-5D-52-C0-00"
            }
        ],
        "DNSServerList" : "192.168.143.1"
}
"@

wsl --shutdown
Get-HnsNetworkEx | Where-Object { $_.Name -Eq "WSL" } | Remove-HnsNetworkEx
#try this, note the reliance on ICS might cause weirdness with assigning static addresses to multiple VMs on the vSwitch
New-HnsNetworkEx -Id B95D0C5E-57D4-412B-B571-18A81A16E005 -JsonString $network
#alternately try:

#this refers to https://github.com/ocroz/wsl2-boot/blob/master/windows/HnsEx.ps1
#  New-HnsNetwork -Name "WSL" -AddressPrefix "192.168.143.0/24" -GatewayAddress "192.168.143.1"
Get-HnsNetworkEx | Where-Object { $_.Name -Eq "WSL" } | convertto-json -depth 100 | set-content c:\users\public\wsl_hcn.json
```

2. create a wsl_conf.sh file

```
$wslconfsh = @'
#!/usr/bin/env bash
echo $time > /var/log/out.out
dev=eth0
currentIP=$(ip addr show $dev | grep 'inet\b' | awk '{print $2}' | head -n 1)
localsubnet=192.168.143
ifaddr=$localsubnet.%%%%%
gateway=$localsubnet.1
ip -4 address flush label $dev
ip addr add $ifaddr/24 broadcast $localsubnet.255 dev $dev
ip route add 0.0.0.0/0 via $gateway dev $dev

'@
```

 
3. create a wsl.conf template to be placed in `/etc/wsl.conf` on the VMs (note that the automatic execution of boot command will work on Windows 11 or server 2022)

```
# refer to the following for more info: https://docs.microsoft.com/en-us/windows/wsl/wsl-config#wsl-2-settings
$wslconf = @'
[automount]
enabled = true
root = /mnt/
options = 'metadata,umask=22,fmask=11'
mountFsTab = true

[network]
hostname = ubuntu_@@@@@
generateHosts = true
generateResolvConf = true

[interop]
enabled = true
appendWindowsPath = false

[user]
default = ubuntu


# "The Boot setting is only available on Windows 11 and Server 2022." - https://learn.microsoft.com/en-us/windows/wsl/wsl-config#boot-settings
[boot]
command = /usr/bin/bash /boot/wsl_conf.sh

'@
```

4. install ubuntu distro and convert to WSL2
* https://github.com/kaisalmen/wsltooling/blob/main/installUbuntuLTS.ps1

```
# distros are available: https://learn.microsoft.com/en-us/windows/wsl/install-manual
mkdir -p $env:userprofile\wsl2\ubuntu\x64
#Invoke-WebRequest -Uri https://aka.ms/wslubuntu -OutFile ubuntu.appx -UseBasicParsing
#or
azcopy copy https://wslstorestorage.blob.core.windows.net/wslblob/Ubuntu2204-221101.AppxBundle $env:userprofile\wsl2\ubuntu.appx

expand-archive $env:userprofile\wsl2\ubuntu.appx $env:userprofile\wsl2\ubuntu
expand-archive $env:userprofile\wsl2\ubuntu\Ubuntu*_x64.appx $env:userprofile\wsl2\ubuntu\x64

#the basic store installation is different: Add-AppxPackage .\ubuntu.appx
wsl --import ubuntu_baseline $env:userprofile\wsl2 $env:userprofile\wsl2\ubuntu\x64\install.tar.gz
wsl --set-version ubuntu_baseline 2
#open docker desktop and enable ubuntu via settings>resources> WSL integration (if you don't see Ubuntu listed, you may need to restart docker desktop)
```

5. power up the ubuntu instance and test
```
#in powershell
wsl -l -v
#start the ubuntu instance
wsl -d ubuntu_baseline hostname
wsl -d ubuntu_baseline hostname -I

#you're still in powershell

#write a /etc/wsl.conf to the target WSL2 VM's disk
101 | % { ( ($wslconfsh -replace "%%%%%","$_") -replace "`r","")  | set-content \\wsl$\ubuntu_baseline\boot\wsl_conf.sh}
($wslconf -replace "@@@@@","baseline") | set-content \\wsl$\ubuntu_baseline\etc\wsl.conf
wsl -d ubuntu_baseline /usr/bin/chmod 744 /boot/wsl_conf.sh
wsl -d ubuntu_baseline ls -al /boot/wsl_conf.sh

#set the IP, without this it appears all VMs get the same IP, as this is functionality of ICS.
wsl -d ubuntu_baseline /boot/wsl_conf.sh

#check the hostname and IP
wsl -d ubuntu_baseline hostname
wsl -d ubuntu_baseline hostname -I
```

6. VERY IMPORTANT POINT: the `[boot]` option in `wsl.conf` is not honored in Windows versions below Windows 11 or Server 2022.
  * The workaround for this is to set the static IP after the WSL2 VM is up and the host compute network vSwitch is (re)configured:
  ```
  wsl -d ubuntu_baseline /boot/wsl_conf.sh
  ```

7. Every time your machine is rebooted, the vSwitch is re-created off of a base template or some logic that I can't locate or can't be customized.

8. In order to configure delete then create the vSwitch as covered in step 1, I've tried a few ways to create a powershell script that invokes HCN vswitch deletion/recreation, but it always fails to start... I've set it to run with highest privs and as the user.  To maintain security of the script file, I've leveraged EFS (local NTFS encryption) to encrypt against the user.  The issue seems separate and may be caused by the NGAV/EDR/EPP I have on my system.

So... in this case, you need to re-run a powershell script with admin privs, to create the vSwitch and issue the commands eery time Windows starts.

The main problem I seem to be having is that I can't assign different IPs to different VMs.  Let's see if I can get through this.


9.  create four ubuntu containers, convert them to be hosted as WSL2 VMs, and create /etc/wsl.conf (https://www.mourtada.se/installing-multiple-instances-of-ubuntu-in-wsl2/)

```
$nodes = "control" #,"workernode1","workernode2","workernode3"
$i=11 #this will be the iterator of the IP address

foreach ($node in $nodes) {
  
  write-host building ubuntu_$node node with WSL host compute network IP ending in $i
  wsl --import ubuntu_$node $env:userprofile\wsl2\ubuntu_$node $env:userprofile\wsl2\ubuntu\x64\install.tar.gz
  
  write-host converting ubuntu_$node node to WSL2 VM
  wsl --set-version ubuntu_$node 2
  write-host "old host info: $(wsl -d ubuntu_$node hostname) at $(wsl -d ubuntu_$node hostname -I)"
 
  sleep 3
  $i | % { ($wslconfsh -replace "%%%%%","$_") -replace "@@@@@",$node | set-content \\wsl$\ubuntu_$node\boot\wsl_conf.sh}
  $i | % { ($wslconf -replace "@@@@@",$node) | set-content \\wsl$\ubuntu_$node\etc\wsl.conf}
  wsl -d ubuntu_$node /usr/bin/chmod 744 /boot/wsl_conf.sh
  wsl -d ubuntu_$node /boot/wsl_conf.sh
  sleep 3

  wsl -t ubuntu_$node
  sleep 3
  write-host "new host info: $(wsl -d ubuntu_$node hostname) at $(wsl -d ubuntu_$node hostname -I)"
  $i++
}

```




* additional items:
```
# update upgrade
wsl -d ubuntu_baseline apt update -y && apt upgrade -y

# maintain a /etc/host file, which you can do from windows 

#note that you can configure RAM, CPU, etc: https://learn.microsoft.com/en-us/windows/wsl/wsl-config
```
