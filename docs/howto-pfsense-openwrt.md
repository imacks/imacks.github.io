Segregate traffic over a single wire
====================================
You have a PFSense firewall and OpenWRT router. You have different types of users who should be subject 
to different firewall rules. All your users should connect to the router instead of directly to the 
firewall.

Your firewall is linked to the router by a **SINGLE** ethernet cable.

The answer? VLAN tagging!


How it works
------------
Every network packet has a header, which stores metadata like MAC address. There is a field in the header for 
storing something called VLAN, which is essentially just a number from 1 to 4096.

Your network applicances can make use of this VLAN field to treat packets coming from a single ethernet 
cable as numerous different virtual cables. In our scenario, the router will process data from any of its 
physical ports, as well as over WiFI radio. We'll make the router tag its packets based on which physical 
port or radio the data originates.


OpenWRT stock configuration
---------------------------
Say your router has 5 physical ports. We'll then see 6 plugs on `network > switch`:

| VLAN ID   | CPU   | LAN1   | LAN2   | LAN3   | LAN4   | WAN   |
|-----------|-------|--------|--------|--------|--------|-------|
| 1         | tag   | untag  | untag  | untag  | untag  | off   |
| 2         | tag   | off    | off    | off    | off    | off   |

OpenWRT uses the switch configuration to associate physical ports on the router to VLANs. In most cases, it 
associate VLAN 2 to your WAN port, and VLAN 1 to any of ports 1-4.

```
{isp}---[2]---{router}---[1]----{user}
```

VLANs are further associated with "interfaces". You can go to the `network > interfaces` page to verify 
how each interface is is associated under "Physical Settings". In our case, interface WAN is attached to VLAN 2 
on switch 0 (expressed as `eth0.2`). Interface LAN is attached to `eth0.1`.


Our configuration
-----------------
Go to `network > switch`. Make sure you check 'Enable VLAN functionality' and **DO NOT** check the 
mirroring options.

We are going to configure this switch like this:

| VLAN ID   | CPU   | LAN1   | LAN2   | LAN3   | LAN4   | WAN   |
|-----------|-------|--------|--------|--------|--------|-------|
| 1         | tag   | off    | off    | off    | off    | off   |
| 2         | tag   | off    | off    | off    | off    | off   |
| 10        | tag   | untag  | untag  | off    | off    | tag   |
| 20        | tag   | off    | off    | off    | off    | tag   |
| 30        | tag   | off    | off    | off    | untag  | tag   |
| 50        | tag   | off    | off    | untag  | off    | tag   |

Our first two lines here essentially disables the original router design. That means interface WAN/WAN6 and 
LAN are now just dummies.

Next, you can connect your upstream firewall to the WAN port of your router. That's going to 
cause traffic to flow to VLAN 10, 20, 30 and 50 (because they are WAN tagged).

Next, create interface VLAN10. Set protocol to DHCP, use bridge interface, and check interface `eth0.10`. Set 
firewall to `unspecified`.

Repeat for interface VLAN20, VLAN30 and VLAN50.

So what happens is if you plug into physical port 1 or 2, your packets will be tagged with VLAN ID 10 by 
the router. Then it can go via the WAN port to your upstream firewall. Similarly, plugging into port 4 will 
make your packets tagged with 30, and port 3 with 50.

Notice that VLAN 20 is not associated with any ports. What that means is you can't get tagged with 20 by 
plugging your device into the router. Instead, we're going to associate VLAN 20 with a wireless radio! To do 
that, add a new wireless radio, and associate with network VLAN20. Now anyone connected to this wifi hotspot 
will have his wireless packets tagged with 20 too.


Upstream firewall
-----------------
You can now configure your upstream firewall rules to filter based on VLAN ID.

Under `interfaces > assignments`, tab on `VLAN` and add a new VLAN. Set the parent interface to the port which 
connects the firewall to your router. VLAN Tag should correspond to the one on OpenWRT, so in our case it is 
10. Go to `Interface Assignments` and add the new VLAN as an interface (use static IP). Next, go to 
`firewall > rules` and add rules for your new VLAN. You will also want to configure your DHCP server for your 
new VLAN too.

Repeat for VLAN 20, 30 and 50.


