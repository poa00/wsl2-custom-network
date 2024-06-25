# wsl2-custom-network
This project contains powershell scripts for creating custom network for WSL2

This project contains code fragments from https://www.powershellgallery.com/packages/HNS/0.2.4

## Prologue
I like many others wasted half a day trying to figure out how to get WSL2 to run properly in Enterprise environment. I was badly mistaken when I thought that there's a configuration option for WSL2's network.

WSL2 chooses subnet from [private network ranges](https://en.wikipedia.org/wiki/Private_network#Private_IPv4_addresses) using "clever" logic that prevents collisions with known private networks. It works great on your home workstation where you're connected directly to your private network. Unfortunately this logic breaks in almost every enterprise network. In most enterprises workstations have no visibility to reserved private network segments - which means that WSL2 has no visibility when it creates subnet configuration. Considering subnet that WSL2 creates is both random and huge (/20), it's extremly likely to eventually collide with existing subnets and prevent user from accessing services. I was actually not able to get it generate valid configuration after numerous reboots.

The second issue is that WSL2 resets network configuration on every reboot. The end result is that network configuration is more or less random. To use network from WSL2 one must allow traffic from and to all network ranges that WSL2 *might* use. This might be acceptable for some users, but most enterprises enforce strict network security policies.

And as you might expect (with Microsoft products), I could find no meaningful documentation on how WSL2 manages the network configuration. I spent half a day reading posts on GitHub only to find that no-one seems to know. There was a lot of discussions on virtual switches and managing network interfaces. After some debugging it became evident that none of the proposed solutions/hacks work reliably enough.

I was just about to quit and revert back to VirtualBox when I came across [a bit of information that pointed to the right direction](https://github.com/microsoft/WSL/discussions/7395). Few google searches later I was convinced that WSL2 uses Host Compute Network (HCN) Microsoft advertised API as public and documentation looked promising. Unfortunately documentation is outdated and missing a lot of fine details. Fortunately HCN api handles requests and responses in JSON format, so I was able to reverse engineer data structures that were require to create WSL2 compatible network configuration.

So here we have it. A script that creates a WSL2 compatible network using Host Computer Network API.

## Usage
Please notice that you most certainly want to edit the sample network configuration. Would you like to use NAT over ICS (google for difference), set Type to NAT and Flags to 0.

I've found that DNSServerList doesn't seem to have any effect. In NAT mode there is no local DNS server, so you need to edit your /etc/resolv.conf and add DNS server that works for your environment.

1. Install WSL2
2. Download contents of this repository
3. Open PowerShell as administractor change to git repository location and execute:
```
import-module -Name .\hcn

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
        "DNSServerList" : "192.168.5.1, 192.168.5.2"
}
"@

Get-HnsNetworkEx | Where-Object { $_.Name -Eq "WSL" } | Remove-HnsNetworkEx
New-HnsNetworkEx -Id B95D0C5E-57D4-412B-B571-18A81A16E005 -JsonString $network
```
4. Once you're happy with the configuration, set up a startup script that creates WSL2 network during startup.


## From comments found in various issues / discussions

### This forked-repo's Issue #2

The following bruit force and no intelligence script works RANDOMLY, but your one is intelligent.
```powershell
echo "Restarting WSL Service"
wsl --shutdown
Restart-Service LxssManager
echo "Restarting Host Network Service"
Stop-Service -name "hns"
Start-Service -name "hns"
 echo "Restarting Hyper-V adapters"
Get-NetAdapter -IncludeHidden | Where-Object `
     {$_.InterfaceDescription.StartsWith('Hyper-V Virtual Switch Extension Adapter')} `
     | Disable-NetAdapter -Confirm:$False
 Get-NetAdapter -IncludeHidden | Where-Object `
     {$_.InterfaceDescription.StartsWith('Hyper-V Virtual Switch Extension Adapter')} `
     | Enable-NetAdapter -Confirm:$False
```

A primary symptom is if you manage to get WSL2 networks working the PC going to sleep breaks it but not other apps networks. 

For corporate environments where IT prevent PowerShell scripts running (even when in an elevated prompt), the repo's PowerShell fix is based off of some C or C++ code from [here](https://github.com/microsoft/WSL/discussions/7395 "WSL Discussion #7395") -- shown below:

### Discussion #7395
#### @pieterhouwen on Nov 13, 2021
Hi there! Can this be used for #7690 ? Is there a way to create a new switch adapter and specify in the wsl2 distro which one to use?

#### @Biswa96 on Nov 13, 2021
No, there can only one virtual network adapter be attached to WSL2 VM. The GUID netId is unique. Bridging network interfaces may have inverse effect.

#### @skorhone on Nov 28, 2021
@Biswa96 My (example) script reconfigures subnet that WSL2 uses using a static network range. This change doesn't have any major side-effects. Unfortunately configuring static IP the right way would require configuring static MAC for WSL2 VM and static MAC to IP rule for DHCP server. All this probably is doable, but reverse-engineering isn't worth the time. If you don't care about possible side-effects, you can set desired static IP by running a script inside the WSL2 vm. If you want to minimize risk of side-effects (IP conflict), prefer IP addresses from end of the network range

#### @Biswa96 on Nov 28, 2021
There is no reverse engineering. Evrything is documented by Microsoft. We've to just complete the puzzle. There will be no IP conflict because there is only one WSL2 VM for a specific PC. To set a specific IP address, it has to be configured first from Windows side. I shall share the sample code after some days.

#### @skorhone on Nov 28, 2021
Microsoft documentation for HCN API is far from complete. I found it easier to extract required information from microsoft/hcsshim than try to spell their cryptic documentation. HCN under the hood uses ICS for assigning IP addresses. I wasn't able to find anything useful for HCN + ICS integration. If you do find related documentation, share it with others 🙂 I'm personally happy with the solution I made. I'm not using Wsl2 for replacing standard VMs. Vagrant + HyperV is much better suited for that. I use Wsl2 mainly as a platform to run tools (podman, python, bash... ) & this use case doesn't warrant static IPs. My only issue with Wsl2 is that we're not able to control network range with a group policy. It'd probably take a day or two from microsoft to implement this ☹️

#### @Biswa96 on Nov 28, 2021
I mean this one https://docs.microsoft.com/en-us/windows-server/networking/technologies/hcn/hcn-top

#### Biswa96 on Dec 14, 2021
⚠️ ⚠️ **The following code is just for testing purposes. Please don't use this for any serious stuff.** ⚠️ ⚠️

> To add a specific IP address in WSL2 VM, follow this code.
> Remember to run this code after WSL2 VM starts.
> This code removes the existing virtual network adapter attached to WSL2 VM and attaches a new one with your provided configuration.
> Run the ip commands in WSL2 shown at the end of the code.

```cpp
#include <windows.h>
#include <computecore.h>
#include <computenetwork.h>

#ifdef _MSC_VER
#pragma comment(lib, "computecore")
#pragma comment(lib, "computenetwork")
#endif

static const GUID NetworkId = {
    0xB95D0C5E,
    0x57D4,
    0x412B,
    { 0xB5, 0x71, 0x18, 0xA8, 0x1A, 0x16, 0xE0, 0x05 } };

static const GUID EndpointId = {
    0x12345678,
    0x1234,
    0x1234,
    { 0x12, 0x34, 0x12, 0x34, 0x56, 0x78, 0x90, 0xAB } };

static const wchar_t NetworkSettings[] = LR"(
    {
        "Name" : "WSL",
        "Type" : "ICS",
        "IsolateSwitch" : true,
        "Flags" : 9,
        "Subnets": [
            {
                "AddressPrefix":"172.16.0.0/24"
            }
        ]
    })";

// VirtualNetwork should be same with NetworkId
// Others are random at your choice
static const wchar_t EndpointSettings[] = LR"(
    {
        "ID": "12345678-1234-1234-1234-1234567890AB",
        "IPAddress": "172.16.0.2",
        "MacAddress": "12-12-12-12-12-12",
        "VirtualNetwork" : "B95D0C5E-57D4-412B-B571-18A81A16E005",
        "Namespace" : {
            "ID" : "00000000-0000-0000-0000-000000000000",
            "IsDefault" : false
        }
    })";

static const wchar_t NicAdapterRemove[] = LR"(
    {
        "ResourcePath" : "VirtualMachine/Devices/NetworkAdapters/Default",
        "RequestType" : "Remove"
    })";

// EndpointId and MacAddress should match with EndpointSettings
static const wchar_t NicAdapterAdd[] = LR"(
    {
        "ResourcePath" : "VirtualMachine/Devices/NetworkAdapters/Default",
        "RequestType" : "Add",
        "Settings" : {
                "EndpointId" : "12345678-1234-1234-1234-1234567890AB",
                "MacAddress" : "12-12-12-12-12-12"
            }
    })";

int main(int argc, char *argv[])
{
    HRESULT hRes;
    HCN_NETWORK HcnNet;
    HCN_ENDPOINT HcnEndp;
    HCS_OPERATION HcsOp;
    HCS_SYSTEM HcsSys;
    PWSTR Result;

    // Put the VmId from `hcsdiag.exe list` command
    wchar_t VmId[] = L"";

    HcsOp = HcsCreateOperation(NULL, NULL);
    hRes = HcsOpenComputeSystem(VmId, GENERIC_ALL, &HcsSys);

    // 1. Delete existing network
    hRes = HcnDeleteNetwork(NetworkId, &Result);

    // 2. Create network and endpoint
    hRes = HcnCreateNetwork(NetworkId, NetworkSettings, &HcnNet, &Result);
    hRes = HcnCreateEndpoint(HcnNet, EndpointId, EndpointSettings, &HcnEndp, &Result);

    // 3. Detach existing network
    hRes = HcsModifyComputeSystem(HcsSys, HcsOp, NicAdapterRemove, NULL);
    hRes = HcsWaitForOperationResult(HcsOp, INFINITE, &Result);

    // 4. Add our own network
    hRes = HcsModifyComputeSystem(HcsSys, HcsOp, NicAdapterAdd, NULL);
    hRes = HcsWaitForOperationResult(HcsOp, INFINITE, &Result);

    hRes = HcnCloseEndpoint(HcnEndp);
    hRes = HcnCloseNetwork(HcnNet);

    HcsCloseOperation(HcsOp);
    HcsCloseComputeSystem(HcsSys);

    return 0;
}

// Now run these commands in WSL2:
// sudo ip address add 172.16.0.2/24 broadcast + dev eth0
// sudo ip link set dev eth0 up
// sudo ip route add default via 172.16.0.1 dev eth0
// Feel free to modify and add this code in your project
```
Also ask any questions here. A proper attribution would be appreciated ❤️

#### @git-rz on Sep 29, 2022
Hi @Biswa96 , I have tried using the code from the original post. It is not working for me.
>Installed Visual Studio 2022
>Created a new c++ command line solution
>Replaced the hello-world code with the code from above
>Modified the address prefix to be 192.168.43.0/24
>Reviewed all my startup programs and removed any that could possibly start wsl
>Added the newly compiled command line tool to my startup folder
>rebooted
>logged in, waited for all startup programs to complete.
>Opened WSL, ran ip addr, saw a 172 address.
I am using w10 enterprise, 21H2 Is there anything else that needs to be adapted to work on my computer?

#### @Biswa96 on Sep 29, 2022
The code in original post is just a proof-of-concept. Try not to use it in enterprise environment ⚠️ The return `HRESULT` values need to be checked. The program should be run as administrator and startup programs do not run as admin. It is not required to reboot PC to modify network. 
1. Shutdown any WSL instance if running.
2. Delete existing WSL2 network interface with this command in Powershell.
```batch
powershell.exe -NoProfile "Get-HnsNetwork | Where-Object {$_.Name -eq 'WSL'} | Remove-HnsNetwork"
```
I shall make a tool with all the modifications that can be done with WSL2 VM. Now I grasped a json parser in C++

#### @davidshen84 on Jan 23, 2023
Hi, could someone help me understand why we have to have a dynamic IP for the WSL instance? I've seen many posts talking about how to set the WSL IP to static. Why this cannot be part of the WSL official feature?

#### @Biswa96 on Jan 23, 2023
Only Microsoft developers can provide proper answer. But recently, there was a feature added to attach any virtual network switch. So, one can set the network range with that. For my usecase, I have a small command line tool with the idea from original post.

#### @pieterhouwen on Jan 23, 2023
Does this mean that you can assign multiple switches or does this just mean that you're not limited to the standard vEthernet switch? Asking in regards to #7395 (comment)

#### @Biswa96 on Jan 23, 2023
I am not sure about that. The options are not officially documented. There are many users using them, see these issues https://github.com/microsoft/WSL/issues?q=is%3Aissue+is%3Aopen+vmswitch

#### @micheldiemer on Apr 18
See down below a detailed working procedure on how to use `WSLAttachSwitch`
Script : #4799 (below)
Text : #4799 (also below)

#### @micheldiemer on Apr 18
See also:

- https://github.com/skorhone/wsl2-custom-network
- networkingMode=mirrored
- ```
  Computer\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Lxss\NatGatewayIpAddress 
  Computer\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Lxss\NatNetwork 
  Computer\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Lxss
  ```

Windows 11 22H2 build 22621 (Pro version at least)
WSL 2.1.5
Create a virtual switch in Hyper-V and use WSLAttachSwitch.exe (may not be possible for you)

```powershell
# RunAs Administrator

# settings
$switchName="StaticIPs"
$submaskCidrBits=24
$winIp="192.168.10.1"
$wslIp="192.168.10.2"
# WSL path for WSLAttachSwitch.exe, will be downloaded via curl
$exeFile="/mnt/c/Windows/System32/WSLAttachSwitch.exe"

# set / check firewal rules
Set-NetFirewallHyperVVMSetting -Name '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}' -DefaultInboundAction Allow
Get-NetFirewallHyperVVMSetting -PolicyStore ActiveStore -Name '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}'

# create a virtual switch and create the IP address of the windows host
New-VMSwitch -SwitchName $switchName -SwitchType Internal
$netadapter=Get-Netadapter | where-Object Name -Like "*$switchName*"
New-NetIPAddress -PrefixLength $submaskCidrBits -InterfaceIndex $netadapter.ifIndex -IPAddress $winIp

# out of scope / optional : connect other HyperV machines to the VMSwitch

# download WSLAttachSwitch
$cmd = "curl -L -o $exeFile https://github.com/dantmnf/WSLAttachSwitch/releases/download/latest/WSLAttachSwitch.exe" 
wsl -u root bash -c $cmd

## UNCOMMENT THIESE LINES IF YOU WANT TO TEST BEFORE MAKING PERMANENT CHANGES
## # attach switch to vm
## $cmd = "sudo $exeFile  $switchName"
## wsl bash -c $cmd
## # assign static IP to vm
## $cmd = "sudo ip addr add $wslIp dev eth1"
## wsl bash -c $cmd
## # all the machines connected to $switchName can ping / communicate with each other

## permanent changes using /etc/rc.local and /etc/wsl.conf
# create or append commands to /etc/rc.local
$cmd = "[ ! -f /etc/rc.local ] && echo '#!/bin/sh -e' > /etc/rc.local"
wsl -u root bash -c $cmd
# assign switch to WSL vm
$cmd = "echo $exeFile $switchName | sudo tee -a /etc/rc.local"
wsl bash -c $cmd
# assign ip address to WSL vm
$cmd = "echo 'ip addr add $wslIp dev eth1' >> /etc/rc.local"
wsl -u root bash -c $cmd
# make sure /etc/rc.local is loaded on boot via /etc/wsl.conf
wsl -u root bash -c "sudo chmod u+x  /etc/rc.local"
wsl -u root bash -c "echo '[boot]' >>  /etc/wsl.conf "
wsl -u root bash -c "echo 'command=/etc/rc.local' >> /etc/wsl.conf "

# Enjoy all machines connected to the switch being able to communicate with each other !
```
#### @micheldiemer commented on Apr 18 
Please find below a more textual description of the above script :

1. Create a Hyper-V virtual switch or use it. Its name is "VirtualSwitch"
2. Adjust firewall rules of WSL
```powershell
Set-NetFirewallHyperVVMSetting -Name '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}' -DefaultInboundAction Allow
Get-NetFirewallHyperVVMSetting -PolicyStore ActiveStore -Name '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}'
```
3. Download [WSLAttachSwitch.exe](https://github.com/dantmnf/WSLAttachSwitch/releases/download/latest/WSLAttachSwitch.exe)
4. on your Windows hard drive (not WSL), e.g. /mnt/c/Windows/System32/WSLAttachSwitch.exe
5. Attach the switch to the WSL vm
6. **File `/etc/rc.local` (must be executable)**
```bash
#!/bin/sh -e
/mnt/c/Windows/System32/WSLAttachSwitch.exe VirtualSwitch
# bonus : assign ip address
# ip addr add 192.168.0.2 dev eth1
File /etc/wsl.conf

[boot]
command=/etc/rc.local
```

Gist in french : https://gist.github.com/micheldiemer/e32a294cd484eff7bcd362bf8f3330b3#file-wsl_hyperv_switch-md
