# Building a VM where it has its own IP address on the local network via ethernet and connecting the host to the network using wifi

## Goal

- Building a VM with separate network identity -so it appears as a device on the local network- to try new things I learn on,for example: like hosting http server and blocking traffic using iptables and try that from my host or ssh or to try VNC or NFS, etc....

## Notes before you start

- You have to be connected to the network through WIFI, because we will use the ethernet for the VM.

- A bridged VM sits on your real LAN, so don’t leave deliberately vulnerable services running on it.

## Environment

  **Host**:Fedora 44 , KVM/QEMU , libvirt , virtual-machine-manager(GUI app)

  **VM**:Ubuntu 24.04 

## Steps

1. Find your ethernet interface name use: `ip -br link` the ethernet interface is the one starting with letter (e).


2. Create the bridge device (virtual switch device).  `sudo nmcli con add type bridge ifname br0 con-name br0 ipv4.method disabled ipv6.method disabled` **Note**:

  * Actually the virtual switch can get an IP address for the host as NetworkManager treats it as normal interface for the host and just send DHCP request and get an IP that's why you see **ipv4.method disabled ipv6.method disabled** in the command,
and actually that's the normal way of doing that and the host and the VM both connects to the network via one ethernet cable,
but for our purpose is to make a separate route for the VM to connect to the network differnet from the host which is connected through wifi.

  



3. Enslave the physical ethernet NIC of your device so it becomes just a port on the virtual switch and that also will make NetworkManager not to ask for an IP using this interface. `sudo nmcli con add type ethernet slave-type bridge con-name br0-port ifname interface-name master br0`


4. `nmcli con up br0-port` just to make sure that the active profile for the ethernet interface is the slave-type bridge u just made.


5. You should shutdown the VM if its on.`sudo virsh shutdown <vm-name>`


6. `sudo virsh attach-interface <vm-name> bridge --source br0 --model virtio --config` to add the bridge interface for the VM and the `--source` flag is just specifying the bridge device -u made in step 2- for the VM to use.


7. `sudo virsh domiflist <vm-name>` which will list to u all the interfaces and their mac-addresses and type(network or bridge).![domiflist](images/domiflist.png)


8. `sudo virsh detach-interface <vm-name> network --mac <mac-address> --config` **NOTE** > From the step 7 table u will get the mac-address(its the mac of the interface which its type is network NOT bridge).


9. Finally, open your VM from the GUI app and check by running `ip addr`.![ip-vm](images/ip-addr-vm.png)

And also you should have different IP on the host.![ip-host](images/ip-addr-host.png)

And you can also run this in host `bridge link show`.![bridge-link-show](images/bridge-link-show.png)


## Undo 

If u wanna undo all this just delete the ethernet NIC profile `sudo nmcli con delete br0-port`
And the virtual switch (br0) profile `sudo nmcli con delete br0` 

And also u will have to detach the NIC which its network source is (bridge) and create another NIC device with default network source (NAT):

* `sudo virsh detach-interface <vm-name> --type bridge --mac <mac-address> --config` ,
u will also get the mac-address from running `sudo virsh domiflist <vm-name>` 

* `sudo virsh attach-interface <vm-name> --type network  --source default --model virtio  --config`


## Explaining everything in details 


**vnet0** : is a tap device created by libvirt(the management layer, it creates and tracks the VMs) to connect it to the virtual switch, think of it like a logical ethernet cable that only exists in RAM.


**br0** : is the virtual switch device we created , it also live in RAM only that's why nmcli creates a profile for it to be able to create it again if u rebooted your host device , u can see that by running `nmcli con show` ![nmcli](images/nmcli-con-show.png)
u will see in the name column (br0) and in the device column (br0) u can actually choose whatever name u want(I personally named it vswitch) , we just name the virtual switch device and its profile-name the same name for simplicity, you can actually see it listed with your other interfaces if you run `nmcli device status` ![nmcli](images/nmcli-device-status.png)

**br0-port** : is the active profile connection for the host's ethernet NIC that makes it act like a socket on the virtual switch so that NetworkManager doesn't ask for an IP (DHCP request) for the host using this interface when that profile is active.
You can also use your ethernet NIC again normally for your host by changing between active profile connections for the ethernet interface.
*when u need the interface to act normally just drop the active profile that makes the interface act like a socket* `nmcli con down br0-port`
And *activate whatever connection u want* `nmcli con up <profile-name>`
**Note**:
   * A bridge with no ports shows an all-zero MAC. With ports it takes the lowest MAC among its ports.
   You will notice this when you change the active profile from the 'slave profile' to something else or you disconnect the host's ethernet interface and
   the VM is off or its network source is NAT(becuase the tap device will be connected even if there is no network connection) you will see all-zero MAC.

--------------------------------------------------------------------------------------------- 
    
In order to make a VM able to have its own IP on the network , we have to somehow  physically connect it to an ethernet socket on a real physical switch device (or your home router as it's a router and a switch in the same time)  but a in this case the device we want to connect the physical switch device is a virtualized environment inside your host , so to solve this problem we have to create a virtual switch inside the kernel so it acts as a bridge that connects virtual environment to a real physical device(switch), and then the VM connects its virtual NIC to that virtual switch using a tap device , then we 'enslave'(making the ethernet NIC of the host just a socket on the virtual switch we created) the ethernet NIC of the host so that when we plug in ethernet cable into the host physical socket and the other end plugged in the physical switch device , now the VM can send data to the physical switch itself and request an IP from the router(DHCP) and when a reply come back the data go thro the host ethernet socket then to the virtual switch we created and at this point the virtual switch see the destination mac address of the VM's virtual NIC and forward the data to the VM .
