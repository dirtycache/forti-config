This document explains how to build a "standard" IPsec tunnel with IPv4 numbered VTI interfaces.

Known to work between FortiGates with v7.2 and lightly tested with other OEM devices, but as always YMMV.

Numbered VTIs are useful because you have a next-hop address which you can ping and use with routing protocols, versus the default unnumbered behavior of "if it matches policy, throw it in the encryption engine and hope it comes out the other side!".

# Configuration Components:

## config vpn ipsec phase1-interface

Defines the phase1 IPsec parameters and binds the tunnel to a specific local interface, usually WAN.

| Variable | Description | Example |
| --- | --- | --- |
| `S{TunnelName}` | A string used to name the phase1-interface, phase2-interface, and logical system interface. I like to keep these consistent.  I suggest using the local interface, destination, and destination interface. In the example below, the name reflects device FOO, port wan1, tunneling to device BAR, wan1.  No spaces, max length 15 characters. | `w1_FOO-w1_BAR` |
| `${TunnelBindIntfc}` | The local interface to which you intend to bind this tunnel. | `wan1` |
| `${Local_IPv4_Endpoint}` | The local device's global source address on the bound interface. Defaults to 0.0.0.0 if unset; this usually works fine when ${TunnelBindIntfc} holds only one IPv4 address. | `192.0.2.1` |
| `${Peer_IPv4_Endpoint}` | The remote device's global address with which you are establishing IPsec. | `198.51.100.44` |
| `${TunnelPSK}`| The pre-shared key for this tunnel | `SuperSecretTunnelPSK!` |

### Example config:

    config vpn ipsec phase1-interface
        edit "S{TunnelName}"
            set interface "${TunnelBindIntfc}"
            set ike-version 2
    	    set local-gw "${Local_IPv4_Endpoint}"
            set peertype any
            set net-device enable
            set proposal aes256-sha512
            set auto-negotiate disable
            set dpd on-idle
            set dhgrp 21
            set nattraversal disable
            set remote-gw ${Peer_IPv4_Endpoint}
            set psksecret ${TunnelPSK}
        next
    end

## config vpn ipsec phase2-interface

Defines the acceptable IPsec phase2 proposals, enables keepalive, and associates to the phase1-interface configuration construct of the same name. By default, FortiGate's phase 2 traffic selector (known to Cisco-heads as "encryption domain"; that is "interesting traffic we want to match and throw over this tunnel" is 0.0.0.0/0. This is a Good Thing. You allow all IPv4 addresses to match the tunnel, and use routing and firewall policy to control addressability and accessibility. Then, when you wish to permit any new src/dst combination of traffic, you can do so without needing to redefine the IPsec phase2 subnets - and bouncing your tunnel. 

| Variable | Description | Example |
| --- | --- | --- |
| `S{TunnelName}` | Same as previous. | `w1_FOO-w1_BAR` |

### Example config:

    config vpn ipsec phase2-interface
        edit "${TunnelName}
            set phase1name "${TunnelName}"
            set proposal aes256-sha512
            set keepalive enable
        next
    end

## config system interface

Instantiates and configures the logical VTI interface

| Variable | Description | Example |
| --- | --- | --- |
| `S{TunnelName}` | Same as previous. | `w1_FOO-w1_BAR` |
| `${Local_VTI_IPv4}` | The IPv4 address which binds to the *local device* VTI interface, aka the "tunnel inside address" - this example uses /31 prefix length for a point to point IPsec VPN. | `203.0.113.2` |
| `${Remote_VTI_IPv4}` | The IPv4 address which binds to the *peer device* VTI interface, aka the "tunnel inside address" - this example uses /31 prefix length for a point to point IPsec VPN. | `203.0.113.3` |
| `${PeerDeviceName}` | Peer device name, used in interface description | `BAR` |
| `${PeerDeviceIntfc}` | The interface on which the tunnel establishes on the peer device, if known. Otherwise omit this from your VTI interface description. | `wan1` |
| `${TunnelBindIntfc}` | Same as previous. This use is to associate the logical VTI interface of the local device with the physical interface applicable to the IPsec tunnel. |

### Example config:

    config system interface
        edit "${TunnelName}"
            set vdom "root"
            set ip ${Local_VTI_IPv4} 255.255.255.255
            set allowaccess ping
            set type tunnel
            set remote-ip ${Remote_VTI_IPv4} 255.255.255.254
            set description "IPsec: ${PeerDeviceName} ${PeerDeviceIntfc}
            set monitor-bandwidth enable
            set role wan
            set interface "${TunnelBindIntfc}"
        next
    end

## config router static (Part 1: VTI)

At least with 7.2 code, FortiGates do not appropriately inject kernel routes for only the locally held IPv4 /32 on the VTI, rather than marking the entire /31 directly connected. Set a static route to fix this problem.

| Variable | Description | Example |
| --- | --- | --- |
| `${VTI_IPv4_Prefix}` | The IPv4 /31 prefix between the local and remote VTIs. | `203.0.113.2/31` |
| `${RouteSeq}` | FortiGate syntax requires each route configuration object to have an integer ID. These are automatically generated by the web GUI, but must be specified when working with the CLI. Please ensure you do not use an ID which is already in use on your device, because you will modify an existing route configuration rather than creating a new one. | 200 |

### Example config:

    config router static
        edit ${RouteSeq}
            set dst ${VTI_IPv4_Prefix}/31
            set device "${TunnelName}"
        next
    end

# A note about IPsec and SD-WAN!

At this point, you may decide to create a new SD-WAN zone. This is useful in a situation where device FOO has 2 ISPs and you wish to create 2 IPsec tunnels to BAR, one over each ISP. Simply put, an SD-WAN zone for IPsec collapses multiple tunnels between the same two peers into a logical grouping (and logical interface, used for firewall policy)

This scenario would change the interface names you use - chiefly, where you would use interface `w1_FOO-w1_BAR` you would instead specify an SD-WAN zone name such as `sdw_FOO-BAR`.

Perhaps a future revision of this guide will include creating such an SD-WAN zone, but for the moment, it does not.

For now, consider that once you build objects and policy referencing any particular interface, it gets much harder to migrate that interface into an SD-WAN zone later. Therefore, consider building an SD-WAN zone even if you only have one member interface.

## config firewall address

Now it's time to construct firewall objects. You'll create at least two `firewall address` objects: one each for IPv4 addresses on both ends of the tunnel. In this case we will use bogon IPv4 space: 10.0.0.0/8 as local to device FOO, and 100.64.0.0/10 as local to remote device BAR. 

I recommend adding a network object for your interface /31 as well.  (`${VTI_IPv4_Prefix}`)

It is acceptable with `set subnet` to use either CIDR notation or dotted quad.

**NOTE: In the example, the option of `set allow-routing enable` is set. This means you can use this named address object in place of a numerical prefix when creating static routes.

### Example config:

    config firewall address
        edit "n-10.0.0.0_8-Bogon_RFC1918"
            set allow-routing enable
            set subnet 10.0.0.0 255.0.0.0
        next
        edit "n-100.64.0.0_10-Bogon_CGNAT"
            set allow-routing enable
            set subnet 100.64.0.0 255.192.0.0
        next
    end

## config firewall addrgrp

Next, we add a layer of abstraction which is useful in policy. We could write the policy referencing the `address` objects directly:

| SrcAddr | DstAddr |
| --- | ---|
| n-10.0.0.0_8-Bogon_RFC1918 | n-100.64.0.0_10-Bogon_CGNAT |

However when you wish to add another prefix as a source or destination, you must modify the policy. Arguably dangerous, or at least more dangerous than the recommended approach: Create **groups**, put things (networks, hosts, other groups) into the group, and reference only groups in your policy.  It is perfectly cromulent for a group to contain only one member. Add multiple members via the GUI, or by concatenating them within the `set member` directive. Quotes are optional, but become mandatory if your group names contain spaces - which is itself not recommended.

In the example config below, the prefix 'gn' indicates it is a group of networks. You may also consider 'gh' for groups of hosts, 'gg' for nesting groups, etc.

As with the firewall address object, the `set allow-routing enable` flag is set so we can use this group object for static routing. **If routing is enabled on a group, it must also be enabled upon all members of the group.**

### Example config:

    config firewall addrgrp
        edit "gn-FOO_Subnets"
            set member "n-10.0.0.0_8-Bogon_RFC1918"
            set allow-routing enable
        next
        edit "gn-BAR_Subnets"
            set member "n-100.64.0.0_10-Bogon_CGNAT" 
            set allow-routing enable
        next
    end

## config router static (Part 2: Remote Subnets)

This is how you tell your local FortiGate FOO about subnets reachable via the tunnel you just made. You'lll add a static route referencing group `gn-BAR-Subnets` via next-hop address of BAR's VTI. **Should you ever need to add an additional prefix, all you have to do is add the address object to the group object - the routing configuration (and firewall policy, as well as anything else that references the group) will inherit and update based upon members of the group.  Cool, huh?**

***NOTE: Notice the syntax for use with an *object* is `set dstaddr` as opposed to `set dst` when routing IP/mask!***

### Example config:

    config router static
        edit ${RouteSeq}
            set dstaddr "gn-BAR_Subnets"
            set device "${TunnelName}"
	    set gateway "${Remote_VTI_IPv4}"
        next
    end

## Sanity Check

If you tunnel is working, you should be able to ping the remote device's VTI with `exec ping ${Remote_VTI_IPv4}`

This does not require any firewall policy itself by default, as the FortiGate considers it local traffic versus forwarded packets. If, however, you have `firewall local-in-policy` configured (it is permissive by default) you may need to add local-in-policy rules to accomodate the /31 used between your VTIs.

## config firewall policy

Now that you have created your tunnel, your address objects, and your address groups, it's time to assemble it all into a firewall policy. This example consists of two polices: 1) permit all IPv4 traffic from `gn-FOO_Subnets` to `gn-BAR_Subnets` and 2) the reverse. It further assumes that on FOO, `gn-FOO_Subnets` prefixes are reachable via local interface `lan`. 

Depending on platform, this could be `lan` or `internal1` or a zone containing several interfaces. **Use your brain. It's your network, after all.**

As with routing objects in the CLI, policy entries require an ID. Again, these are generated automatically when using the FortiGate GUI. Use care on the CLI to not clobber existing policy. If you create via CLI, you may wish to re-order using the GUI - it's an absolute pain in the ass to re-order via CLI.

### Example config:

    config firewall policy
        edit 38
            set name "IPsec: FOO to BAR"
            set srcintf "lan"
            set dstintf "w1_FOO-w1_BAR"
            set action accept
            set srcaddr "gn-FOO_Subnets"
            set dstaddr "gn-BAR_Subnets"
            set schedule "always"
            set service "ALL"
            set logtraffic all
        next
        edit 39
            set name "IPsec: BAR to FOO"
            set srcintf "w1_FOO-w1_BAR"
            set dstintf "lan"
            set action accept
            set srcaddr "gn-BAR_Subnets"
            set dstaddr "gn-FOO_Subnets"
            set schedule "always"
            set service "ALL"
            set logtraffic all
        next
    end

## Conclusion

If all has gone according to plan, you should now have a functional IPsec tunnel between devices, with an IPv4 numbered VTI interface.
