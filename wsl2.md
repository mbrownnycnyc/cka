* remember to run `wsl --update` to update the kernel on the VM distro.

* wait, so let me backup...
  * I always thought WSL2 was actually a VM host.  It is, but it also generates a single VM per distro, and by distro I mean "ubuntu".
  * So you get a single ubuntu VM, with various things shared between... each container!
  * Each container (what MSFT seems to like to call "lightweight linux utility VM") has several things unique to that container itself.
  * One of those things is network namespaces.  https://github.com/microsoft/WSL/issues/4304#issuecomment-511884889
* So, this is why every time I change the IP of "my WSL2 VM instance" it changes... well... the IP of my WSL2 VM instance... NOT THE CONTAINER.
  * For lack of knowledge, I'm going to just say that the networking structure of the containers is transparent, and the vSwitch interface "eth0" on the containers, has an IP bound.  You can actively change this IP given MANY solutions:
    * https://github.com/wikiped/WSL-IpHandler <-- simple to use and easy to understand
    * https://github.com/ocroz/wsl2-boot <-- complex to use and hard to understand
    * https://github.com/skorhone/wsl2-custom-network <-- okay, but incomplete (I think?)
* What I really want to do is build a k8s cluster of "VMs" on WSL2 so I can progress my training [and, NO... at this point I'm going down the ship... no other hypervisor for me until I tackle _this_ challenge... why?  so I can understand linux features and containers better... is this not _really_ the goal anyway?]
  * Now I can do this by "tricking" WSL2 to host multiple VMs... which I'm certainly not sure how to do.
  * Or I can try to use network namespaces (netns) to create a single network namespace per container.
    * Here's some guidance on that:
      * concepts: https://blogs.igalia.com/dpino/2016/04/10/network-namespaces/
      * concepts and implementation: https://superuser.com/a/1715457/91174
        * or https://gist.github.com/mbrownnycnyc/145dacb59a0cb73ebebd8bc1ff5d5033
    * In this, I have two obvious requirements:
      * Attach an IP address to each WSL2 **container** that allows it to be accessible to:
        * other **containers** running within the WSL2 VM distro (aka on the same vSwitch).
        * The host OS networking stack via the "WSL" vSwitch.
* okay, LET'S GOOOOOO
* investigate: https://github.com/rootless-containers/slirp4netns

1. instantiate a new ubuntu distro based ~~lightweight linux utility VM~~ WSL2 VM hosted container.
```
# reference: https://github.com/kaisalmen/wsltooling/blob/main/installUbuntuLTS.ps1
# distros are available: https://learn.microsoft.com/en-us/windows/wsl/install-manual
mkdir -p $env:userprofile\wsl2\ubuntu\x64
#Invoke-WebRequest -Uri https://aka.ms/wslubuntu -OutFile ubuntu.appx -UseBasicParsing
#or get azcopy https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-v10#download-azcopy
azcopy copy https://wslstorestorage.blob.core.windows.net/wslblob/Ubuntu2204-221101.AppxBundle $env:userprofile\wsl2\ubuntu.appx

expand-archive $env:userprofile\wsl2\ubuntu.appx $env:userprofile\wsl2\ubuntu
expand-archive $env:userprofile\wsl2\ubuntu\Ubuntu*_x64.appx $env:userprofile\wsl2\ubuntu\x64

#update to the latest kernel
wsl --update

wsl --import ubuntu_baseline $env:userprofile\wsl2 $env:userprofile\wsl2\ubuntu\x64\install.tar.gz
wsl --set-version ubuntu_baseline 2
#open docker desktop and enable ubuntu via settings>resources> WSL integration (if you don't see Ubuntu listed, you may need to restart docker desktop)


```

2. now we have a container accessible via `wsl -d ubuntu_baseline`, so let's build a net network namespace
```
wsl -d ubuntu_baseline

# Create veth link.
ip link add v-eth1 type veth peer name v-peer1

# set the IP of the virtual interface that will provide 0/0 route (we will use 10.200.1.0/24)
ip addr add 10.200.1.1/24 dev v-eth1
ip link set v-eth1 up


#list the namespaces (there aren't any)
ip netns

#create a namespace
ip netns add netns1

# Add peer-1 to NS.
ip link set v-peer1 netns netns1

#set the IP of the "local" interface (for this WSL2 container)
ip netns exec netns1 ip addr add 10.200.1.2/24 dev v-peer1
ip netns exec netns1 ip link set v-peer1 up

# add a loopback and bring it up
ip netns exec netns1 ip link set dev lo up
ip netns exec netns1 ip link set lo up

# add a gateway of last resort into the routing table to exit via the virtual ethernet
ip netns exec netns1 ip route add default via 10.200.1.1


#Share internet access between host and NS.
# Enable IP-forwarding.
echo 1 > /proc/sys/net/ipv4/ip_forward

# Flush forward rules, policy DROP by default.
iptables -P FORWARD DROP
iptables -F FORWARD

# Flush nat rules.
iptables -t nat -F

# Enable masquerading of 10.200.1.0 (aka NATing)
iptables -t nat -A POSTROUTING -s 10.200.1.0/255.255.255.0 -o eth0 -j MASQUERADE

# Allow forwarding between eth0 and v-eth1.
iptables -A FORWARD -i eth0 -o v-eth1 -j ACCEPT
iptables -A FORWARD -o eth0 -i v-eth1 -j ACCEPT


#verify you can ping an outside IP from network namespace netns1
ip netns exec netns1 ping 8.8.8.8

#verify the routing table within netns1
ip netns exec netns1 ip route sh

```

3. route traffic destined for 10.200.1.0/24 to the WSL vSwitch interface:
```
#must be run as elevated
New-NetRoute -DestinationPrefix "10.200.1.0/24" -interfacealias "vEthernet (WSL)" -NextHop 10.200.1.1
```

4. the IP addressing will not stay persistent through WSL2 VM reboots

5. Build another WSL2 container and take a look at the namespace:

```

```




# initial setup

https://askubuntu.com/questions/1435938/is-it-possible-to-run-a-wsl-app-in-the-background

1. download the ubuntu distro ~~lightweight linux utility VM~~ WSL2 VM hosted container.
```
# reference: https://github.com/kaisalmen/wsltooling/blob/main/installUbuntuLTS.ps1
# distros are available: https://learn.microsoft.com/en-us/windows/wsl/install-manual
mkdir -p $env:userprofile\wsl2\ubuntu\x64
#Invoke-WebRequest -Uri https://aka.ms/wslubuntu -OutFile ubuntu.appx -UseBasicParsing
#or get azcopy https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-v10#download-azcopy
azcopy copy https://wslstorestorage.blob.core.windows.net/wslblob/Ubuntu2204-221101.AppxBundle $env:userprofile\wsl2\ubuntu.appx

expand-archive $env:userprofile\wsl2\ubuntu.appx $env:userprofile\wsl2\ubuntu
expand-archive $env:userprofile\wsl2\ubuntu\Ubuntu*_x64.appx $env:userprofile\wsl2\ubuntu\x64

#update to the latest kernel
wsl --update
```

2. create a wsl.conf file for each of the target containers:
```
# refer to the following for more info: https://docs.microsoft.com/en-us/windows/wsl/wsl-config#wsl-2-settings
$wslconf = @'
[automount]
enabled = true
root = /mnt/
options = 'metadata,umask=22,fmask=11'
mountFsTab = true

[network]
hostname = @@@@@
generateHosts = true
generateResolvConf = true

[interop]
enabled = true
appendWindowsPath = false

[user]
#default = ubuntu


# "The Boot setting is only available on Windows 11 and Server 2022." - https://learn.microsoft.com/en-us/windows/wsl/wsl-config#boot-settings
[boot]
command = /usr/bin/bash /boot/netns_conf.sh

'@
```

3. create a netns_conf.sh file that we can execute on target containers every time the WSL2 container host VM restarts:

```
$netshconfsh = @'
#!/usr/bin/env bash

if [ ! -d /sys/class/net/v-eth1 ]; then
  # Create veth link.
  ip link add v-eth1 type veth peer name v-peer1

  # set the IP of the virtual interface that will provide 0/0 route (we will use 10.200.1.0/24)
  ip addr add 10.200.1.1/24 dev v-eth1
  ip link set v-eth1 up
fi

#create a namespace
ip netns add netns1

# Add peer-1 to NS.
ip link set v-peer1 netns netns1

#set the IP of the "local" interface (for this WSL2 container)
ip netns exec netns1 ip addr add 10.200.1.%%%%%/24 dev v-peer1
ip netns exec netns1 ip link set v-peer1 up

# add a loopback and bring it up
ip netns exec netns1 ip link set dev lo up
ip netns exec netns1 ip link set lo up

# add a gateway of last resort into the routing table to exit via the virtual ethernet
ip netns exec netns1 ip route add default via 10.200.1.1


#Share internet access between host and NS.
# Enable IP-forwarding.
echo 1 > /proc/sys/net/ipv4/ip_forward

# Flush forward rules, policy DROP by default.
iptables -P FORWARD DROP
iptables -F FORWARD

# Flush nat rules.
iptables -t nat -F

# Enable masquerading of 10.200.1.0 (aka NATing)
iptables -t nat -A POSTROUTING -s 10.200.1.0/255.255.255.0 -o eth0 -j MASQUERADE

# Allow forwarding between eth0 and v-eth1.
iptables -A FORWARD -i eth0 -o v-eth1 -j ACCEPT
iptables -A FORWARD -o eth0 -i v-eth1 -j ACCEPT

#create an listener which stops the container from shutting down (https://askubuntu.com/a/1436045/514844)
ip netns exec netns1 nohup nc -l 127.0.0.1 8000 &

'@
```

4. container builds: (must be run within user for which you'll be accessing the WSL2 VM hosted containers)
```
#ref: https://www.mourtada.se/installing-multiple-instances-of-ubuntu-in-wsl2/
$nodes = "ubuntu_control" ,"ubuntu_workernode1","ubuntu_workernode2","ubuntu_workernode3"
$i=111 #this will be the iterator of the IP address

foreach ($node in $nodes) {
  
  write-host building $node node with WSL2 netns IP ending in $i
  wsl --import $node $env:userprofile\wsl2\$node $env:userprofile\wsl2\ubuntu\x64\install.tar.gz
  
  write-host converting $node container to WSL2 VM hosted container
  wsl --set-version $node 2
  sleep 3
  $i | % { ($wslconf -replace "@@@@@",$node) | set-content \\wsl$\$node\etc\wsl.conf}
  wsl -t $node
  sleep 3
  wsl -d $node hostname
  $i | % { ($netshconfsh -replace "%%%%%","$_") | set-content \\wsl$\$node\boot\netns_conf.sh}
  wsl -d $node /usr/bin/chmod 744 /boot/netns_conf.sh
  sleep 3
  $i++
}
```

* Given the following, I think I need to build all containers, power them all on, and then instantiate the `veth`:
```
root@ubuntu_control:/mnt/c/Users/MBrown# ip -o link show | grep v-eth
14: v-eth1@if13: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000\    link/ether 2e:08:fa:f2:73:48 brd ff:ff:ff:ff:ff:ff link-netns netns1
root@ubuntu_control:/mnt/c/Users/MBrown# ip netns
netns1 (id: 0)

#notice the netns id (and the presence of `link-netnsid 0`... there is no associated netns to the veth... the peer interface exists but isn't accessible, so you recieve an error when invoking `ip link set v-peer1 netns netns1` 'Cannot find device "v-peer1"')
root@ubuntu_workernode1:/mnt/c/Users/MBrown# ip -o link | grep v-eth
14: v-eth1@if13: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000\    link/ether 2e:08:fa:f2:73:48 brd ff:ff:ff:ff:ff:ff link-netnsid 0
root@ubuntu_workernode1:/mnt/c/Users/MBrown# ip netns
netns1
```
* Should loop after the containers are all up:
```
wsl -d $node /boot/netns_conf.sh
```

1. add a static route to the Windows VM host routing table:  Since our focus is just providing network access to the containers hosted on the WSL2 VM, we don't care if the WSL2 VM has a static IP, so all of the complexity in managing that challenge is removed:
```
#this must be executed from an elevated prompt
#this will survive reboots of WSL2 container host VM as well as your Windows host system:
New-NetRoute -DestinationPrefix "10.200.1.0/24" -interfacealias "vEthernet (WSL)" -NextHop 10.200.1.1
```

1. add entries to hosts file on Windows host: (must be executed elavated)

```
$nodes = "ubuntu_control" ,"ubuntu_workernode1","ubuntu_workernode2","ubuntu_workernode3"
$i=111
foreach ($node in $nodes) {
  $i | % { "10.200.1.$i $($node).local" | add-content C:\Windows\System32\drivers\etc\hosts}
  $i++
}
```


7. from the windows host, check routes:

```
$nodes = "ubuntu_control" ,"ubuntu_workernode1","ubuntu_workernode2","ubuntu_workernode3"
foreach ($node in $nodes) {
  
  fastping $node
}

```
   
8. upon reboot of WSL2 VM (after a `wsl --shutdown` or Windows host system reboot), you must perform the following:

```
$nodes = "ubuntu_control" ,"ubuntu_workernode1","ubuntu_workernode2","ubuntu_workernode3"
wsl -d $node /boot/netns_conf.sh
```







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
#ref: https://www.mourtada.se/installing-multiple-instances-of-ubuntu-in-wsl2/
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
