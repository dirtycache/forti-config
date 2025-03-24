This document explains how to build a "standard" SSLVPN on a FortiGate given the following scenario:

- VPN has an FQDN record which resolves in global Internet DNS
- SSLVPN listening interface is a loopback, and client connections are made via wan1 + wan2 and VIPs
- SSLVPN listening port is 443/tcp
- SSL certificate is obtained automatically by ACME from Let's Encrypt. 
- Uses a named address specified IPv4 client address pool 
- SAML authentication (FortiGate is the SP) of users via SAML IdP (such as Duo SSO or Entra Identity)
- FortiGate group matching IdP group
- Policy which permits access from connected clients in a specified group to a "LAN subnets" which is attached to FortiGate interface (zone) "Zone_LAN"
- Web acess is completely disabled, including replacemsg

Known to work with FortiGates with v7.2 firmware and free version of FortiClient for Windows, Mac, and iOS.  As always, YMMV.

Variables used are in FortiManager script syntax.

# Variables Table

| Variable | Description | Example |
| --- | --- | --- |
| `$(VPNFQDN)` | The FQDN of the SSLVPN connection. | vpn.example.com |
| `$(ACME_Email)` | The email address used for Let's Encrypt certificates | ssl@example.com |
| `S{IntfcACME}` | The FortiGate interface to which the ACME client is associated, such as wan1, wan2, port1, port2, etc. | `wan1` |
| `${srcAddrACME}` | The IPv4 address used for the ACME client source. Defaults to 0.0.0.0 (the interface primary IP address) if unset. | `203.0.113.42` |
| `$(IntfcACME_NextHop)` | The IPv4 next-hop (default gateway toward global Internet) for `($IntfcACME)` | `203.0.113.1` |
| `$(IntfcAddr_wan2)` | The IPv4 address attached to the interface of your second internet connection, e.g., `wan2` | `203.0.113.200` |
| `$(loopback_VPN_IPv4_Addr` | The IPv4 address, /32, bound to the SSLVPN loopback. This example uses an arbitrary address within reserved CGNAT space. You may use almost any address you like, even to the point of one consistent address across multiple firewalls you control. It is not shared, not routed, and only used internally by the VIPs. | `100.93.42.1` |
| `$(LAN_Prefix)` | The IPv4 prefix for this firewall's LAN | `198.51.100.0` |
| `$(LAN_CIDR)` | The CIDR netmask for `$(LAN_Prefix)` - number only, no slash | `24` |
| `$(SSLVPN_Client_Prefix)` | The IPv4 prefix for addresses assigned to SSLVPN clients. | `192.0.2.0` |
| `$(SSLVPN_Client_Prefix_CIDR)` | The CIDR netmask for `$(SSLVPN_Client_Prefix)` - number only, no slash | `24` |
| `$(SSLVPN_SAML-SP-Name)` | A name given to your SAML SP | `Duo_App-VPN` or `Entra-SSO` |
| `$(VPN_SAML-SP-CertName` | A name string given to the object of the SAML SP's certificate. This will always be `REMOTE_Cert_1` (or 2, 3...) if GUI certificate import is used. | `Cert-Rem_VPN_SAML-SP` |
| `$(VPN_SecGroup)` | The name of the security group containing users permitted to access this SSLVPN. This group will exist both on your FortiGate and within your SAML IdP; I highly recommend making them the same value. | `SEC-SSLVPN_Users` | 
| `$(ClientDNS1)` | The IP address of a DNS resolver you wish SSLVPN clients to query. | `198.51.100.10` |
| `$(ClientDNS2)` | The IP address of a DNS resolver you wish SSLVPN clients to query. | `198.51.100.11` |

# Configuration Steps:

## 1) bind a secondary IP address to the ACME interface

Bind the secondary /32 address upon `$(srcIntfcACME)` which is used as the ACME client source-ip. 

Binding to the interface allows the FortiGate to listen for and respond to `http-01` challenges to port 80/tcp. 

*A secondary address is not explicitly required, but I think it's a good idea when one is available.*

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

## 2) define named address objects for Let's Encrypt

Add address objects for hosts `*.letsencrypt.org`, `letsencrypt.org`. Next, create a group of these hosts.

### config script:

    config firewall address
        edit "h-FQDN-WILDCARD.letsencrypt.org"
            set type fqdn
            set comment "ACME operations"
            set allow-routing enable
            set fqdn "*.letsencrypt.org"
        next
        edit "h-FQDN-letsencrypt.org"
            set type fqdn
            set comment "ACME operations"
            set allow-routing enable
            set fqdn "letsencrypt.org"
        next
    end

    config firewall addrgrp
        edit "gh-LetsEncryptACME"
            set member "h-FQDN-WILDCARD.letsencrypt.org" "h-FQDN-letsencrypt.org"
            set comment "ACME operations"
            set allow-routing enable
        next
    end

## 3) add static route for Let's Encrypt

This step routes ACME traffic via `$(IntfcACME)` - this is required in an SD-WAN scenario because `config system acme` does not support use of SD-WAN.

### config script:

    config router static
          edit 0
            set status enable
            set comment 'ACME certificate operations'
            set device "$(IntfcACME)"
            set gateway $(IntfcACME_NextHop)
            set distance 10
            set dstaddr "gh-LetsEncryptACME"
            set comment "ACME operations"
        next
    end

## 4) Ensure the ACME server resolves and is reachable from source address $(srcAddrACME)

### runtime commands:

    # execute ping-options source $(srcAddrACME)
    # execute ping acme-v02.api.letsencrypt.org

If you do not receive a successful ping response, stop here and troubleshoot until you achieve a good response.
You may wish to run `# diagnose sniffer packet any 'icmp and host 172.65.32.248' 4 0 l` to ensure the request and response are both observed via the intended interface.

## 5)  Configure certificate request

### config script:

    config vpn certificate local
        edit "$(VPN_FQDN)"
            set enroll-protocol acme2
            set acme-domain "$(VPN_FQDN)"
            set acme-email "$(ACME_Email)"
        next

    (the next two lines are output returned from the CLI - do not paste!)
    By enabling this feature you declare that you agree to the Terms of Service at https://acme-v02.api.letsencrypt.org/directory`
    Do you want to continue? (y/n)

        y
    end

## 6)	Verify the certificate enrollment was successful

### runtime commands:

`# get vpn certificate local details $(VPNFQDN)`

Output should look like:

    # get vpn certificate local details vpn.example.com
    == [ vpn.example.com ]
    Name:        vpn.example.com
    Subject:     CN = vpn.example.com
    Issuer:      C = US, O = Let's Encrypt, CN = R11
    Valid from:  <date>
    Valid to:    <date>
    Fingerprint: <fingerprint>
    Serial Num:  <serial number>
    ACME details:
    Status: The certificate for the managed domain has been renewed successfully and can be used (valid since Wed, 05 Mar 2025 20:18:04 GMT).
    Staging status: Nothing in staging

## 7)	Create the firewall internal loopback interface to which SSLVPN will be bound.

### config script:

    config system interface
        edit "loopback_VPN"
            set vdom "root"
            set ip $(loopback_VPN_IPv4_Addr)/32
            set allowaccess ping
            set type loopback
            set role wan
        next
    end

## 8) Create standardized firewall address and firewall addrgrp objects on this firewall (these may already exist in your environment)

### config script:

This defines the LAN subnet.

    config firewall address
        edit "n-$(LAN_Prefix)_$(LAN_CIDR)-LAN"
            set allow-routing enable
            set subnet $(LAN_Prefix)/$(LAN_CIDR)
        next
    end

This creates an address group to contain LAN subnets, with the previous address object as a member.

    config firewall addrgrp
       edit "gn-LAN_Subnets"
            set member "n-$(LAN_Prefix)_$(LAN_CIDR)-LAN" 
            set allow-routing enable
        next
    end

This creates an address object for the SSLVPN client address pool.

    config firewall address
        edit "n-$(SSLVPN_Client_Prefix)_$(SSLVPN_Client_Prefix_Mask)-SSLVPN"
            set allow-routing enable
            set subnet 192.168.243.0/24
        next
    end

## 9) Adjust remote authentication timeout from 5 seconds (default) to 60 seconds to accomodate the SAML iDP and user interaction

### config script:

    config system global
        set remoteauthtimeout 60
    end

## 10) Firewall static routing

I like to do this as a "doublecheck" that the SSLVPN Client subnet is routed always via the FortiGate's `ssl.root` interface.

### config script:

    config router static
        edit 0
            set comment "$(VPNFQDN) client pool"
            set distance 10
            set dstaddr  "n-$(SSLVPN_Client_Prefix)_$(SSLVPN_Client_Prefix_Mask)-SSLVPN"
            set device "ssl.root"
        next
    end

## 11) Upload the IdP certificate to the firewall

After you configure SAML between your FortiGate and the remote SP (such as a Duo Application, Entra Enterprise App SSO, or any other) you should be prompted to download that entity's SSL certificate. You'll need to add this to your FortiGate configuration. There are two ways to do this:

    - Upload via the FortiGate GUI via System -> Certificates -> Import Remote
    - Paste certificate text into a config script and execute from the CLI

I prefer the latter of these options because it allows you to specify the name of the imported certificate. The GUI method will immutably name the certificate `REMOTE_Cert_1` (or 2 or 3 if any already exist on the system). **Insert your own certifcate data where indicated in the script!** Do not terminate the double quote of the `set remote` line until after the end of the certificate data.a

If you're using FortiManager, this could be a great thing to define as a metadata variable per device, perhaps `$(VPN_SP_CertData)` but you'd have to find some way of shoveling the data over from the SP cert download into the FMG.

### config script:

    config vpn certificate remote
        edit "$(VPN_SAML-SP-CertName"
            set remote "-----BEGIN CERTIFICATE-----
            blah-blah-gluk-gluk-certificate-base64
            blah-blah-gluk-gluk-certificate-base64
            blah-blah-gluk-gluk-certificate-base64
            blah-blah-gluk-gluk-certificate-base64
            blah-blah-gluk-gluk-certificate-base64
            blah-blah-gluk-gluk-certificate-base64
            blah-blah-gluk-gluk-certificate-base64
            blah-blah-gluk-gluk-certificate-base64
            blah-blah-gluk-gluk-certificate-base64
            blah-blah-gluk-gluk-certificate-base64
            blah-blah-gluk-gluk-certificate-base64
            blah-blah-gluk-gluk-certificate-base64
            blah-blah-gluk-gluk-certificate-base64
            blah-blah-gluk-gluk-certificate-base64
            blah-blah-gluk-gluk-certificate-base64
            blah-blah-gluk-gluk-certificate-base64
            blah-blah-gluk-gluk==
            -----END CERTIFICATE-----"
    next
    end

## 12)  Configure the saml user on your FortiGate:

NOTE: You will need certain URLs of the SP to configure SAML with your FortiGate, and possibly username and group name attribute claims in this configuration.

Such specifics are outside the scope of this guide.

### config script:

    config user saml
        edit "$(SSLVPN_SAML-SP-Name)"
            set cert "$(VPNFQDN)"
            set entity-id "https://$(VPNFQDN)/remote/saml/metadata"
            set single-sign-on-url "https://$(VPNFQDN)/remote/saml/login"
            set single-logout-url "https://$(VPNFQDN)/remote/saml/logout"
            set idp-entity-id "<your_IdP_entity_id>"
            set idp-single-sign-on-url "<your_IdP_login_url>"
            set idp-single-logout-url "<your_IdP_logout_url>"
            set idp-cert "$(VPN_SAML-SP-CertName"
            set digest-method sha1
        next
    end

## 13) Group matching

You typically need to configure group matching between groups existent within your SAML IdP. This is only an example - refer to documentation of your SAML provider. The important thing to note here is that `set server-name` within the stanza of `config match -> edit 0` is that `server-name` refers to the name of the object you created in the previous step under `config user saml` e.g., "`SSLVPN-SP`".

### config script:

    config user group
        edit "$(VPN_SecGroup)"
            set member "$(SSLVPN_SAML-SP-Name)"
                config match
                    edit 1
                        set server-name "$(SSLVPN_SAML-SP-Name)"
                        set group-name "$(VPN_SecGroup)"
                    next
                end
        next
    end

## 14)  Configure VIP - wan1

Configure the address for the VIP on `wan1`. Or `port1` or `x2` or whatever you have used if your FortiGate doesn't have an interface named `wan1`. Whatever, you do you - it's your network!  Here, we assume "your first Internet circuit == `wan1`".

This is the same IPv4 address you specified previously for the ACME client source-ip (`${srcAddrACME})`), and **may** be a secondary /32 on `wan1` if you have one available and configured, or it might be the only IPv4 address configured on `wan1`. Either scenario will work.

Notice that there are two VIPs to configure. The first is for port 443/tcp and the second for 4433/udp (DTLS). Configuring these VIPs is necessary because the one-to-one port forwarding syntax of the VIP configuration allows specification of only one protocol/port. The omission of `set protocol` in the first VIP config is simply because the FortiGate default is `set protocol tcp` is implied.

Finally, yes, the `set extip` IP address value is not wrapped in double quotes, and the `set mappedip` value is. No one knows why, that's just how the FortiGate config works, at least in v7.2.

### config script:

    config firewall vip
        edit "VIP-VPN_$(srcAddrACME)_TCP-443"
            set extip $(srcAddrACME)
            set mappedip "$(loopback_VPN_IPv4_Addr)"
            set extintf "any"
            set portforward enable
            set extport 443
            set mappedport 443
        next
        edit "VIP-VPN_$(srcAddrACME)_UDP-4433"
            set extip $(srcAddrACME)
            set mappedip "$(loopback_VPN_IPv4_Addr)"
            set extintf "any"
            set portforward enable
            set protocol udp
            set extport 4433
            set mappedport 4433
        next
    end

## 15) Configure VIP - wan2

Repeat the previous step to configure VIPs on `wan2` if applicable.

### config script:

    config firewall vip
        edit "VIP-VPN_$(IntfcAddr_wan2)_TCP-443"
            set extip $(IntfcAddr_wan2)
            set mappedip "$(loopback_VPN_IPv4_Addr)"
            set extintf "any"
            set portforward enable
            set extport 443
            set mappedport 443
        next
        edit "VIP-VPN_$(IntfcAddr_wan2)_UDP-4433"
            set extip $(IntfcAddr_wan2)
            set mappedip "$(loopback_VPN_IPv4_Addr)"
            set extintf "any"
            set portforward enable
            set protocol udp
            set extport 4433
            set mappedport 4433
        next
    end

## 16) firewall address objects

Create two `firewall address` objects. This allows you to reference the destination L3 IP addresses, rather than the L4 protocol/port VIPs, later on when you configure `firewall policy` and keeps the policy rules more streamlined.

### config script:

    config firewall address
        edit "h-$(srcAddrACME)"
            set allow-routing enable
            set subnet $(srcAddrACME)/32
        next
        edit "h-$(IntfcAddr_wan2)"
            set allow-routing enable
            set subnet $(IntfcAddr_wan2)/32
        next
    end

## 17) firewall addrgrp

Create a `firewall addrgrp` containing the two addresses you created in the previous step.

This allows you to craft one policy rule to permit connections to both VIP adddresses.

### config script:

    config firewall addrgrp
        edit "gh-VPN_VIPs"
            set member "h-$(srcAddrACME)" "h-$(IntfcAddr_wan2)"
            set allow-routing enable
        next
    end

## 18) firewall service custom 

Create `firewall service custom` “DTLS” for 4433/udp.

### config script:

    config firewall service custom
        edit "DTLS"
            set category "Remote Access"
            set udp-portrange 4433
        next
    end

## 19) SSLVPN Portals

Configure the firewall SSLVPN portals. First, `$(VPNFQDN_portal)` for when a user is a member of `$(VPN_SecGroup)` and second, `no-access` for all other users and groups.

### config script:

    config vpn ssl web portal
        edit "$(VPNFQDN_portal)"
            set tunnel-mode enable
            set auto-connect enable
            set keep-alive enable
            set save-password enable
            set ip-pools "«SSLVPNClientPoolName»"
        next
        edit "no-access"
            set forticlient-download disable
        next
    end

## 20) SSLVPN Settings

Configure the firewall SSLVPN settings:

A note about `$(ClientDNS1)` and `$(ClientDNS2)`: These may be any DNS server you wish, but generally is either located on your LAN, or possibly you've attached a /32 from `$(SSLVPN_Client_Prefix)` to your `ssl.root` interface and are going to enable the FortiGate's DNS service on `ssl.root` and use DNS Filter! In that case, you only need `$(ClientDNS1)`<g> 

### config script:

    config vpn ssl settings
        set status enable
        set ssl-min-proto-ver tls1-1
        set servercert "$(VPNFQDN)"
        set tunnel-ip-pools "$(SSLVPN_Client_Prefix)"
        set dns-suffix "example.com"
        set dns-server1 $(ClientDNS1)"
        set dns-server2 $(ClientDNS2)"
        set port 443
        set source-interface "loopback_VPN"
        set source-address "all"
        set source-address6 "all"
        set default-portal "no-access"
        config authentication-rule
            edit 1
                set groups "$(VPN_SecGroup)"
                set portal "$(VPNFQDN_portal)"
            next
    end
end

## 21)	system replacemsg sslvpn sslvpn-login

Configure the SSLVPN Login replacement message to completely disable web mode in favor of FortiClient.

### config script:

    config system replacemsg sslvpn "sslvpn-login"
    set buffer "<!DOCTYPE html>
    <html lang=\"en\" class=\"main-app\">
     <head>
     <meta charset=\"UTF-8\">
     <meta http-equiv=\"X-UA-Compatible\" content=\"IE=8; IE=EDGE\">
     <meta name=\"viewport\" content=\"width=device-width, initial-scale=1\">
     <meta name=\"apple-itunes-app\" content=\"app-id=1475674905\">
     <link href=\"/styles.css\" rel=\"stylesheet\" type=\"text/css\">
     <link href=\"/css/legacy-main.css\" rel=\"stylesheet\" type=\"text/css\">
     <title>
     VPN
     </title>
     </head>
     <body>
     <div class=\"view-container\">
     <form class=\"prompt legacy-prompt\" action=\"%%SSL_ACT%%\" method=\"%%SSL_METHOD%%\" name=\"f\" autocomplete=\"off\">
     <div class=\"content with-header with-sslvpn\">
     <div class=\"sub-content sub-sslvpn\">
     <div class=\"wide-inputs\">
     <p>
     SSL-VPN access via this web portal has been disabled.
     </p>
     <p>
     Please connect using FortiClient.
     </p>
     </div>
     <div class=\"button-actions wide sslvpn-buttons\">
     <button id=\"launch-forticlient-button\" type=\"button\" onClick=\"launchFortiClient()\">
     <f-icon class=\"ftnt-forticlient\">
     </f-icon>
     <span>
     Launch FortiClient
     </span>
     </button>
     <iframe id=\"launch-forticlient-iframe\" style=\"display:none\">
     </iframe>
     <button id=\"saml-login-bn\" class=\"primary\" type=\"button\" name=\"saml_login_bn\" onClick=\"launchSamlLogin()\" style=\"display:none\">
     SSO Login
     </button>
     </div>
     </div>
     </div>
     </form>
     </div>"
    end

## 22) config firewall policy

Add firewall policies.

NOTE: You will need to enable multiple interface policies (found in GUI under Feature Visibility) for these policies to work.

You may also enable it via CLI:

### config script:

    config system settings
        set gui-multiple-interface-policy enable
    end

NOTE: If you wish to connect to your SSLVPN from inside your LAN (hairpin NAT), add the FortiGate's appropriate LAN interface to the `set srcintf` directive in each policy.

After these policies are created, you may wish to reorder them relative to your existing firewall policy - this is best accomplished in the GUI. 

First, permit inbound traffic to your VIPs. 

NOTE: This policy rule includes service `PING` because this is required for ping response to work as expected - because your SSLVPN binding is to `loopback_VPN`, yet the packet ingresses via `wan1` or `wan2`, the packet is *traversing the firewall engine* before becoming local traffic to the loopback interface. Thus, policy to permit is required, even if you have no `firewall local-in-policy` enabled.

### config script:
 
 config firewall policy
    edit 0
        set name "VPN VIP ingress"
        set status enable
        set srcintf "wan1" "wan2"
        set dstintf "any"
        set action accept
        set srcaddr "all"
        set dstaddr "gh-VPN_VIPs"
        set schedule "always"
        set service "DTLS" "HTTPS" "PING" "TRACEROUTE"
        set global-label "SSLVPN"
    next
end

Next, add a policy to permit traffic matching the the source: (User in `$(VPN_SecGroup)` +  srcip=`$(SSLVPN_Client_Prefix`) and destination=`$(LAN_Prefix)` via outgoing interface (zone) `Zone_LAN`.

This policy permits all IP traffic. You may wish to craft more granular policies.

### config script:

config firewall policy
    edit 0
        set name "SSLVPN Client Access"
        set status enable
        set srcintf "ssl.root"
        set dstintf "Zone_LAN"
        set action accept
        set srcaddr "$(SSLVPN_Client_Prefix"
        set dstaddr "$(LAN_Prefix)"
        set schedule "always"
        set service "ALL"
        set groups "$(VPN_SecGroup)"
        set global-label "SSLVPN"
    next
end

## 24)	Test and verify proper operation of SSLVPN.


