# PowerVSConnectivity
  <b>Establish connectivity on-premise to PowerVS via IBM Cloud Classic VSRX Gateway</b>

We will realize follwing scenario described in IBM Cloud Documentation
(https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-network-architecture-diagrams#network-reference-architecture-privateipsec)
This is very common scenario and it can be used like general guide for connectivity with IBM Cloud via Ipsec VPN using Juniper VSRX Gateway Appliance.
In many situations need Private connectivity from on-Premise environment to Cloud Resources, less expensive option is VPN.
IBM cloud have different zones like modern MZR, classic infrastructure and Power VS colo.
From network perspective need additional actions to interconnect it from one to each other, which is possible via Transit Gateway (TGW) and via Gateway appliance deployed in Classic infrastructure. 

In our cases, main objective was to establish bi-directional connectivity from client on-premise location to PowerVS colo.
It is possible via two options 
 1. On-Premise GW -VPN GW in VPC + TGW via GRE to PowerVS cloud connection 
 2. On-Premise GW - GW appliance + GRE via Power VS cloud connection
 
<strong>Option 1</strong>
imitations:
Need to enable NAT-T on on-premise Gatawey because it is actual limitation of VPC VPN GW
benefits:
no need to manage
HA pair out of the box
low cost

<strong>Option 2</strong> 
limitations:
Need to order and setup GW appliance
Need to manage it after provisioning 
benefits:
Full control of FW policies and routing

Preffered option in our case was Option - 2 (more flexibility in configuration, ability to use GW appliance like central router and firewall between all services deployed on IBM Cloud Account)
  
![PowerVS-to-on-Premise-Architecture](https://github.com/notras/PowerVSConnectivity/blob/main/GREIpsecPowerVS-GRE.drawiov2.png)

<b> Prerequisites before you start</b>
 1. On-premise device to terminate VPN traffic from IBM cloud.
 2. Plan your network requirements for IBM Power VS subnets, how many, choose CIDR prefixes to not overlap with you local subnet etc
 3. Have at least some experience with Juniper Gateway appliance or simmilar devices 
 4. IBM Cloud account and permissions to provision, and manage services
 5. Estimate your charges for required resources Power VS (billed hourly), GW appliance (billed monthly)  etc and get necessary approvals
 6. VRF and Cloud Endpoints should be enabled on your IBM Cloud account

<b>1 step</b>
Create Power VS service insrtance
https://cloud.ibm.com/catalog/services/power-systems-virtual-server

![Creating PowerVS instance](https://github.com/notras/PowerVSConnectivity/blob/main/powerVSinstanceceration.png)

<b>2 step </b>
Create subnets inside PowerVS service instance, the main reason, it should exist before you order Direct Link 2.0 connection from PowerVS colo to the rest IBM cloud resources, the main reason proper BGP routing.
![Creating Private Network](https://github.com/notras/PowerVSConnectivity/blob/main/PrivateNWcreation.png)
<b> 3 step </b>
We need to order direct link which will connect PowerVS with IBM Cloud services and attach previously created subnet 10.2.2.0/24 or several subnets to the Cloud Connection. IBM provide free of charge option for interconnection between PowerVS Colo and rest of the IBM Cloud services. By default PowerVS ASN 64999 and local IBM Cloud ANS 13884, you can find this details in Interconnectivity section:
https://cloud.ibm.com/interconnectivity when your link will be provisioned.
![Creating Direct Link PowerVS to IBM Cloud](https://github.com/notras/PowerVSConnectivity/blob/main/DirectLinkPower.png)
<b> 3 Step </b> 
Provisioning GW appliance https://cloud.ibm.com/gen1/infrastructure/provision/gateway
You can choose bandwidth, specific version based on your needs, better to deploy this GW in the same Cloud location where PowerVS located, but if you enabled global routing for Direct link it is not mandatory.
![GW Appliance provisioning(]https://github.com/notras/PowerVSConnectivity/blob/main/GWprovisioning.png)
Provisioning took up to four hours.
When GW ready, you will receive email, or you can check it in the portal here: https://cloud.ibm.com/netsec/gateway-appliances
You will see following details (I was replaced real IP's for security reasons)
![GW Appliance settings](https://github.com/notras/PowerVSConnectivity/blob/main/VSRXConfig.png)

You need to record VSRX private IP and Public IP, you will use it for configuration purposes in our case:
Private IP 10.75.12.11
Public IP 161.32.44.198

<b> 4 Step </b>
You have choice to establish VPN from on premise to GW appliance or to establish GRE via Cloud Connection, the results will be the same. We will establish GRE firstly.
We have cloud connection in established state it is point to point connectivity. IBM usually allocate 169.254.0.1/30 on PowerVS router side and 169.254.0.2/30 on the opposite side of IBM Cloud which is all other IBM Cloud Services.
![Cloud Connection](https://github.com/notras/PowerVSConnectivity/blob/main/CloudConnectionSettings.png)
In the virtual connection section you can manage which services will be connected, you can connect only Classic where Juniper provisioned or you can also attach VPC.
You need to enable GRE 
![Virtual connection](https://github.com/notras/PowerVSConnectivity/blob/main/virtualconnectionSettings.png)
And add following GRE settings:
![GRE settings](https://github.com/notras/PowerVSConnectivity/blob/main/GREsettings.png)
GRE destination should be Juniper VSRX Private IP 10.75.12.11
GRE subnet 172.16.2.1/30 ( This is overlay subnet which will be used for P2P communication via GRE between VSRX and PowerVS router which is managed by IBM, you can choose any subnet based on your preferences, you will use IP from this subnet for source and Destination of your GRE tunnel)
We will asign following IP's
PowerVS router IP 172.16.2.5 (if you unable to identify this IP the best option to raise ticket for support and get confirmation about asigned IP)
VSRX GRE IP 172.16.2.6 ( this IP you can assign yourself from the subnet which you selected for GRE)
When you finished configuration on the Cloud connection side, you need to configure VSRX as well. 
<b> 5 Step </b>
GRE configuration on the VSRX
You can choose how to configure VSRX via GUI or via ssh:
Via gui you can connect to private or public VSRX IP from GW configuration page: in our case it will be https://10.75.12.11:8443 to connect via Private IP you need to allow VPN access for your user and enable Motion Pro client VPN. additional details here (https://cloud.ibm.com/docs/iaas-vpn?topic=iaas-vpn-standalone-vpn-clients#macos-standalone-client)
We will connect via ssh with credentials available on the VSRX configuration page
First off all you need to create gre tunnel with following parameters:
Source IP 10.75.12.11 (VSRX private IP)
Destination 172.16.2.1 (GW of overlay subnet which you defined in virtual connection for Cloud Connection we defined 172.16.2.0/30 subnet)
GRE interface IP address on VSRX 172.16.2.6/30
Below how necessary commands
```shell
set interfaces gr-0/0/0 unit 0 tunnel source 10.75.12.11
set interfaces gr-0/0/0 unit 0 tunnel destination 172.16.2.1
set interfaces gr-0/0/0 unit 0 family inet mtu 1400
set interfaces gr-0/0/0 unit 0 family inet address 172.16.2.6/30
```
respective configuration in gui would be:
```
    }
    gr-0/0/0 {
        unit 0 {
            tunnel {
                source 10.75.12.11;
                destination 172.16.2.1;
            }
            family inet {
                mtu 1400;
                address 172.16.2.6/30;
            }
        }
    }
```
next you need to allow security zones, because this traffic flow via private network we allow all, but you can permit only explicit subnets etc if you need for security reason. We allow traffic between Power Zone which is belong to gre interface and allow to traverse traffic to IBM Cloud Classic which is SL-PRIVATE zone by default in IBM Cloud, also for troubleshooting allowed embedded zone junos-host to allow ping from local VSRX interfaces.

```shell
set security zones security-zone POWER interfaces gr-0/0/0.0 host-inbound-traffic system-services all
set security zones security-zone POWER interfaces gr-0/0/0.0 host-inbound-traffic protocols all
set security policies from-zone SL-PRIVATE to-zone POWER policy Private-net-PowerGRE match source-address any
set security policies from-zone SL-PRIVATE to-zone POWER policy Private-net-PowerGRE match destination-address any
set security policies from-zone SL-PRIVATE to-zone POWER policy Private-net-PowerGRE match application any
set security policies from-zone SL-PRIVATE to-zone POWER policy Private-net-PowerGRE then permit
set security policies from-zone POWER to-zone SL-PRIVATE policy PowerGRE-Private-net match source-address any
set security policies from-zone POWER to-zone SL-PRIVATE policy PowerGRE-Private-net match destination-address any
set security policies from-zone POWER to-zone SL-PRIVATE policy PowerGRE-Private-net match application any
set security policies from-zone POWER to-zone SL-PRIVATE policy PowerGRE-Private-net then permit
set security policies from-zone POWER to-zone vpn policy PowerGRE-to-DR match source-address any
set security policies from-zone POWER to-zone vpn policy PowerGRE-to-DR match destination-address any
set security policies from-zone POWER to-zone vpn policy PowerGRE-to-DR match application any
set security policies from-zone POWER to-zone vpn policy PowerGRE-to-DR then permit
set security policies from-zone POWER to-zone POWER policy PowerGRE-to-PowerGRE match source-address any
set security policies from-zone POWER to-zone POWER policy PowerGRE-to-PowerGRE match destination-address any
set security policies from-zone POWER to-zone POWER policy PowerGRE-to-PowerGRE match application any
set security policies from-zone POWER to-zone POWER policy PowerGRE-to-PowerGRE then permit
set security policies from-zone POWER to-zone junos-host policy allow-ping-from-power match source-address any
set security policies from-zone POWER to-zone junos-host policy allow-ping-from-power match destination-address any
set security policies from-zone POWER to-zone junos-host policy allow-ping-from-power match application junos-ping
set security policies from-zone POWER to-zone junos-host policy allow-ping-from-power then permit
set security policies from-zone junos-host to-zone POWER policy allow-ping-to-power match source-address any
set security policies from-zone junos-host to-zone POWER policy allow-ping-to-power match destination-address any
set security policies from-zone junos-host to-zone POWER policy allow-ping-to-power match application junos-ping
set security policies from-zone junos-host to-zone POWER policy allow-ping-to-power then permit
```
In gui it will looks following:
```
from-zone SL-PRIVATE to-zone POWER {
            policy Private-net-PowerGRE {
                match {
                    source-address any;
                    destination-address any;
                    application any;
                }
                then {
                    permit;
                }
            }
        }
        from-zone POWER to-zone SL-PRIVATE {
            policy PowerGRE-Private-net {
                match {
                    source-address any;
                    destination-address any;
                    application any;
                }
                then {
                    permit;
                }
            }
        }
        from-zone POWER to-zone POWER {
            policy PowerGRE-to-PowerGRE {
                match {
                    source-address any;
                    destination-address any;
                    application any;
                }
                then {
                    permit;
                }
            }
        }
        from-zone POWER to-zone junos-host {
            policy allow-ping-from-power {
                match {
                    source-address any;
                    destination-address any;
                    application junos-ping;
                }
                then {
                    permit;
                }
            }
        }
        from-zone junos-host to-zone POWER {
            policy allow-ping-to-power {
                match {
                    source-address any;
                    destination-address any;
                    application junos-ping;
                }
                then {
                    permit;
                }
            }
        }
        security-zone POWER {
            interfaces {
                gr-0/0/0.0 {
                    host-inbound-traffic {
                        system-services {
                            all;
                        }
                        protocols {
                            all;
                        }
                    }
                }
            }
        }
```

Now we need to configure BGP
The BGP neighbor the same IP which is used for GRE destination on the Power VS router end 172.16.2.5
The Power VS ASN 64999
VSRX ASN 64880
```shell
set protocols bgp group tunGRE local-as 64880
set protocols bgp group tunGRE neighbor 10.12.248.1
set protocols bgp group PowerFra type external
set protocols bgp group PowerFra local-address 172.16.2.6
set protocols bgp group PowerFra family inet unicast
set protocols bgp group PowerFra family inet6 unicast
set protocols bgp group PowerFra export adver-prefix
set protocols bgp group PowerFra peer-as 64999
set protocols bgp group PowerFra local-as 64880
set protocols bgp group PowerFra neighbor 172.16.2.5
set routing-options autonomous-system 64880
```
We need to add policy to advertize prefixes from on-premise to Power VS router
```shell
set policy-options policy-statement adver-prefix term 1 from route-filter 10.6.22.0/24 exact
set policy-options policy-statement adver-prefix term 1 from route-filter 10.5.11.0/24 exact
set policy-options policy-statement adver-prefix term 1 then accept
set policy-options policy-statement adver-prefix term default then reject
```
Then we need to allow BGP protocol in VSRX firewall
```shell
set firewall filter PROTECT-IN term BGP from source-address 172.16.2.1/32
set firewall filter PROTECT-IN term BGP from source-address 10.12.248.2/32
set firewall filter PROTECT-IN term BGP from source-address 10.12.248.1/32
set firewall filter PROTECT-IN term BGP from source-address 169.254.0.1/32
set firewall filter PROTECT-IN term BGP from source-address 192.168.4.56/32
set firewall filter PROTECT-IN term BGP from source-address 172.16.2.5/32
set firewall filter PROTECT-IN term BGP from destination-address 169.254.0.1/32
set firewall filter PROTECT-IN term BGP from destination-address 172.16.2.1/32
set firewall filter PROTECT-IN term BGP from destination-address 169.254.0.2/32
set firewall filter PROTECT-IN term BGP from destination-address 10.75.12.11/32
set firewall filter PROTECT-IN term BGP from destination-address 10.12.248.2/32
set firewall filter PROTECT-IN term BGP from destination-address 172.16.2.6/32
set firewall filter PROTECT-IN term BGP from protocol tcp
set firewall filter PROTECT-IN term BGP from port bgp
set firewall filter PROTECT-IN term BGP then accept
```
Respective configuration in another notation are following

```
        }
        group PowerFra {
            type external;
            local-address 172.16.2.6;
            family inet {
                unicast;
            }
            family inet6 {
                unicast;
            }
            export adver-prefix;
            peer-as 64999;
            local-as 64880;
            neighbor 172.16.2.5;
        }
```
```
policy-options {
    policy-statement adver-prefix {
        term 1 {
            from {
                route-filter 10.6.22.0/24 exact;
                route-filter 10.5.11.0/24 exact;
            }
            then accept;
        }
        term default {
            then reject;
        }
    }
}
```

```
policy-options {
    policy-statement adver-prefix {
        term 1 {
            from {
                route-filter 10.6.22.0/24 exact;
                route-filter 10.5.11.0/24 exact;
            }
            then accept;
        }
        term default {
            then reject;
        }
    }
}
```
```
 }
        term BGP {
            from {
                source-address {
                    172.16.2.1/32;
                    10.12.248.2/32;
                    10.12.248.1/32;
                    169.254.0.1/32;
                    192.168.4.56/32;
                    172.16.2.5/32;
                }
                destination-address {
                    169.254.0.1/32;
                    172.16.2.1/32;
                    169.254.0.2/32;
                    10.75.12.11/32;
                    10.12.248.2/32;
                    172.16.2.6/32;
                }
                protocol tcp;
                port bgp;
            }
            then accept;
```

To check that GRE working as expected you can run following command:
If BGP established GRE in operational mode

```shell
show bgp summary
```
```
admin@gateway01-vsrx-vSRX> show bgp summary
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 3 Peers: 3 Down peers: 2
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0              
                       1          1          0          0          0          0
inet6.0              
                       0          0          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
172.16.2.5            64999       3172       3217       0       1  1d 0:06:00 Establ
```
If route advertized, you will then able to route this subnets to VPN tunnel

```shell
show route advertising-protocol bgp 172.16.2.5
```
```
admin@gateway01-vsrx-vSRX> show route advertising-protocol bgp 172.16.2.5

inet.0: 18 destinations, 18 routes (17 active, 0 holddown, 1 hidden)
  Prefix            Nexthop          MED     Lclpref    AS path
* 10.5.11.0/24            Self                                    I
* 10.6.22.0/24            Self                                    I
```

<b> 6 Step </b>
We need to establish VPN to on premise and route traffic from GRE to VPN interface st.0 
I will cover only configuration on the IBM Cloud side, on-premise configuration will be the same and only difference from specific device which used in the organization.

the VSRX Public IP:  161.32.44.198 
the on-premise VPN GE IP: 182.11.38.1
you need to replace pre-shared key "XXXXXXXXXXXXXXXX" to yours

```shell
set security ike proposal VPN-DR-IBM authentication-method pre-shared-keys
set security ike proposal VPN-DR-IBM dh-group group14
set security ike proposal VPN-DR-IBM authentication-algorithm sha-256
set security ike proposal VPN-DR-IBM encryption-algorithm aes-256-cbc
set security ike proposal VPN-DR-IBM lifetime-seconds 86400
set security ike policy VPN-DR-IBM mode main
set security ike policy VPN-DR-IBM reauth-frequency 0
set security ike policy VPN-DR-IBM proposals VPN-DR-IBM
set security ike policy VPN-DR-IBM pre-shared-key ascii-text "XXXXXXXXXXXXXXXX"
set security ike gateway VPN-DR-IBM ike-policy VPN-DR-IBM
set security ike gateway VPN-DR-IBM address 182.11.38.1
set security ike gateway VPN-DR-IBM no-nat-traversal
set security ike gateway VPN-DR-IBM local-identity inet 161.32.44.198
set security ike gateway VPN-DR-IBM remote-identity inet 182.11.38.1
set security ike gateway VPN-DR-IBM external-interface ae1
set security ike gateway VPN-DR-IBM local-address 161.32.44.198
set security ike gateway VPN-DR-IBM version v2-only
set security ike gateway VPN-DR-IBM fragmentation disable
set security ipsec proposal VPN-DR-IBM protocol esp
set security ipsec proposal VPN-DR-IBM authentication-algorithm hmac-sha-256-128
set security ipsec proposal VPN-DR-IBM encryption-algorithm aes-256-cbc
set security ipsec proposal VPN-DR-IBM lifetime-seconds 3600
set security ipsec policy VPN-DR-IBM perfect-forward-secrecy keys group14
set security ipsec policy VPN-DR-IBM proposals VPN-DR-IBM
set security ipsec vpn VPN-DR-IBM bind-interface st0.0
set security ipsec vpn VPN-DR-IBM df-bit clear
set security ipsec vpn VPN-DR-IBM ike gateway VPN-DR-IBM
set security ipsec vpn VPN-DR-IBM ike no-anti-replay
set security ipsec vpn VPN-DR-IBM ike ipsec-policy VPN-DR-IBM
set security ipsec vpn VPN-DR-IBM establish-tunnels immediately
```
in the different notation
```shell
ike {
        proposal VPN-DR-IBM {
            authentication-method pre-shared-keys;
            dh-group group14;
            authentication-algorithm sha-256;
            encryption-algorithm aes-256-cbc;
            lifetime-seconds 86400;
        }
        policy VPN-DR-IBM {
            mode main;
            reauth-frequency 0;
            proposals VPN-DR-IBM;
            pre-shared-key ascii-text "XXXXXXXXXXXXXXXXXXXX"; ## SECRET-DATA
        }
        gateway VPN-DR-IBM {
            ike-policy VPN-DR-IBM;
            address 182.11.38.11;
            no-nat-traversal;
            local-identity inet 161.32.44.198;
            remote-identity inet 182.11.38.11;
            external-interface ae1;
            local-address 161.32.44.198;
            version v2-only;
            fragmentation {
                disable;
            }
        }
    }
    ipsec {
        proposal VPN-DR-IBM {
            protocol esp;
            authentication-algorithm hmac-sha-256-128;
            encryption-algorithm aes-256-cbc;
            lifetime-seconds 3600;
        }
        policy VPN-DR-IBM {
            perfect-forward-secrecy {
                keys group14;
            }
            proposals VPN-DR-IBM;
        }
        vpn VPN-DR-IBM {
            bind-interface st0.0;
            df-bit clear;
            ike {
                gateway VPN-DR-IBM;
                no-anti-replay;
                ipsec-policy VPN-DR-IBM;
            }
            establish-tunnels immediately;
        }
    }

```

Very useful guide from my coleague: https://cloudguy.ca/2022/03/19/connecting-to-ibm-power-systems-virtual-servers-through-direct-link/

IBM cloud documentation:
PowerVS:
https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-network-architecture-diagrams#network-reference-architecture-pvs2pvs
https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-cloud-connections#configure-gre-tunnel
JuniperVSR configuration:
https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-cloud-connections#configure-gre-tunnel
