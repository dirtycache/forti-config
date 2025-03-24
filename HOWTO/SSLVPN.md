This document explains how to build a "standard" SSLVPN on a FortiGate given the following scenario:

- VPN has an FQDN record which resolves in global Internet DNS
- SSLVPN listening interface is a loopback, and client connections are made via wan1 + wan2 and VIPs
- SSL certificate is obtained automatically by ACME from Let's Encrypt. This relies on a secondary IP address bound to wan1 so the FortiGate may listen on 80/tcp for ACME http-01 challenges.
- Uses a named address specified IPv4 client address pool 
- SAML authentication (FortiGate is the SP) of users via Duo SSO. This config may be easily extended to authenticate Entra or other SAML iDP credentials
- FortiGate group matching iDP group
- Policy which permits access from connected clients in a specified group to a "LAN subnets" which is attached to FortiGate interface (zone) "Zone_LAN"
- Web acess is completely disabled, including replacemsg

Known to work with FortiGates with v7.2 firmware and free version of FortiClient for Windows, Mac, and iOS.  As always, YMMV.

# Configuration Steps:

## 1) config system interface

Bind the secondary /32 address upon $(srcIntfcACME) which is used as the ACME client source-ip. Binding to the interface allows the FortiGate to listen for http-01 challenges to port 80/tcp. A secondary address is not explicitly required, but I think it's a good idea when one is available.

| Variable | Description | Example |
| --- | --- | --- |
| `S{IntfcACME}` | The FortiGate interface to which the ACME client is associated, such as wan1, wan2, port1, port2, etc. | `wan1` |
| `${srcAddrACME}` | The IPv4 address used for the ACME client source. Defaults to 0.0.0.0 (the interface primary IP address) if unset. | `203.0.113.42` |

### config script:

    config system interface
        edit "$(IntfcACME)"
            set secondary-IP enable
            config secondaryip
                edit 0
                    set ip $(srcAddrACME)/32
                    set allowaccess ping
                next
            end
        next
    end

### example config:

    config system interface
        edit "wan1"
            set secondary-IP enable
            config secondaryip
                edit 0
                    set ip 203.0.113.42/32
                    set allowaccess ping
                next
            end
        next
    end


