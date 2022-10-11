# PowerVSConnectivity from on-premise via VPN
  <b>Establishing connectivity from on-premise to PowerVS via IBM Cloud Classic VSRX Gateway</b>

We will realize follwing scenario described in IBM Cloud Documentation
(https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-network-architecture-diagrams#network-reference-architecture-privateipsec)
This is very common scenario and it could be useful like general guide for connectivity with IBM Cloud via Ipsec VPN using Juniper VSRX Gateway Appliance in advance to IBM Cloud documentaion.
In many situations need Private connectivity from on-Premise environment to Cloud Resources, the ess expensive option is VPN.
IBM cloud have different zones like modern MZR, classic infrastructure and Power VS colo.
From network perspective need additional actions to interconnect this services inside IBM Cloud from one to each other, which is possible via Transit Gateway (TGW) and via Gateway appliance deployed in Classic infrastructure. 

In our case the main objective was to establish bi-directional VPN connectivity from client on-premise location to PowerVS colo.
It is possible at least via two options 
 1. On-Premise GW -VPN GW in VPC + TGW via GRE to PowerVS cloud connection 
 2. On-Premise GW - GW appliance + GRE via Power VS cloud connection
 
<strong>Option 1</strong>
<p>limitations:</p>

   1. Need to enable NAT-T on on-premise Gateway because it is actual limitation of VPC VPN GW (https://cloud.ibm.com/docs/vpc?topic=vpc-vpn-limitations)
   2. Not full control of VPN tunnel from log perspective and necessary settings
 
 <p>benefits:</p>
 1. no need to manage
 2. HA pair out of the box
 3. low cost

<strong>Option 2</strong> 
limitations:
 1. Need to order and setup GW appliance
 2. Need to manage GW after provisioning
 3. extra cost

<p>benefits:</p>

 1. Full control of FW policies and routing
 2. Ability to use the same GW for control all connections inside IBM Cloud
 3. Higher throughtput up to 10Gbps
 
<p>Preffered option in our case was Option - 2 (more flexibility in configuration, ability to use GW appliance like central router and firewall between all services deployed on IBM Cloud Account)</p>
The overall architecture described below on the diagram, we have on-premise site with two private /24 subnets which is should be advertized to IBM Cloud Power VS systems, in IBM cloud we will deploy several PowerVS VM's in one Private subnet 10.2.2.0/24 with AIX which is suitable for example to hosting Oracle DB, SAP HANA, WAS, Message Broker and many others enterprise SW. IT team from on-premise should able to manage SW stack installed on this VM's and perform replication with on-premise systems for DR purposes with ability to failover users from on-premise backend systems to systems deployed in IBM Cloud via the same VPN.
  
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
https://cloud.ibm.com/catalog/services/power-systems-virtual-server it is just shell for your Power infrastrucuture in specific cloud DC location. Creation of service instance will not produce any cost.

![Creating PowerVS instance](https://github.com/notras/PowerVSConnectivity/blob/main/powerVSinstanceceration.png)

<b>2 step </b>
Create subnets inside PowerVS service instance, the main reason, it should exist before you order Direct Link 2.0 connection from PowerVS colo to the rest IBM cloud resources, the main reason proper BGP routing. We created one subnet 10.2.2.0/24

![Creating Private Network](https://github.com/notras/PowerVSConnectivity/blob/main/PrivateNWcreation.png)

<b> 3 step </b>
We need to order direct link which will connect PowerVS with IBM Cloud services and attach previously created subnet 10.2.2.0/24 or several subnets to the Cloud Connection. 
<p>IBM provide free of charge option for interconnection between PowerVS Colo and rest of the IBM Cloud services.</p>
By default PowerVS ASN 64999 and local IBM Cloud ANS 13884, you can find this details in Interconnectivity section:
https://cloud.ibm.com/interconnectivity when your link will be provisioned. You can also change BGP parameters if it is necessary after connection have been created

![Creating Direct Link PowerVS to IBM Cloud](https://github.com/notras/PowerVSConnectivity/blob/main/DirectLinkPower.png)

<p>Cloud Connection creation is fully automated you will see green status when it will be completed successfully.</p>

<b> 3 Step </b> 
Provisioning of GW appliance https://cloud.ibm.com/gen1/infrastructure/provision/gateway
You can choose bandwidth, specific version based on your needs, better to deploy this GW in the same Cloud location where PowerVS located, but if you enabled global routing for Direct link it is not mandatory.
<p>You can provision single GW or HA pair depending from your requirements</p>
<p>In this scenario we will use single GW apliance:</p>

![GW Appliance provisioning](https://github.com/notras/PowerVSConnectivity/blob/main/GWprovisioning.png)

Provisioning took up to four hours.

When GW ready, you will receive email, or you can check it in the portal here: https://cloud.ibm.com/netsec/gateway-appliances
You will see following details (I was replaced real IP's for security reasons)

![GW Appliance settings](https://github.com/notras/PowerVSConnectivity/blob/main/VSRXConfigv1.png)

You need to record VSRX private IP and Public IP, you will use it for configuration purposes in our case this parameteres are following:
<p> Private IP 10.75.12.11 </p>
<p> Public IP 161.32.44.198 </p>

<b> 4 Step </b>
You have choice to establish VPN from on premise to GW appliance or to establish GRE via Cloud Connection, the final results will be the same. We will establish GRE firstly.
<p>We have cloud connection in established state it is point to point connectivity.</p>
<p>IBM usually allocate 169.254.0.1/30 on PowerVS router side and 169.254.0.2/30 on the opposite side of IBM Cloud.</p>

![Cloud Connection](https://github.com/notras/PowerVSConnectivity/blob/main/CloudConnectionSettings.png)
In the virtual connection section you can manage which services will be connected, you can connect only Classic where Juniper provisioned or you can also attach VPC. For our scenario VPC connectivity not necessary. 

<p> In the virtuall connection settings you need to enable GRE like described  on the screenshot below</p>

![Virtual connection](https://github.com/notras/PowerVSConnectivity/blob/main/virtualconnectionSettings.png)
You need to add following GRE settings:

![GRE settings](https://github.com/notras/PowerVSConnectivity/blob/main/GREsettings.png)

<p>GRE destination should be Juniper VSRX Private IP 10.75.12.11</p>

<p>GRE subnet 172.16.2.0/29 ( This is overlay subnet which will be used for P2P communication via GRE between VSRX and PowerVS router which is managed by IBM, you can choose any subnet based on your preferences, you will use IP from this subnet for source and Destination of your GRE tunnel)</p>
<p>In the IBM Cloud docs provided following recommendations https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-cloud-connections#gre-configuration-example</p>
<p>We will asign following IP's</p>
<p>PowerVS router IP 172.16.2.5 (this IP not visible in UI for your cloud connection and you can not retrieve it via "ibmcloud pi con conID" for now the best option to raise ticket to IBM Cloud support and get confirmation about asigned IP from GRE subnet)</p>
<p>VSRX GRE IP 172.16.2.6 ( this IP you can assign yourself from the subnet which you selected for GRE)</p>

<p>When you finished configuration on the Cloud connection side, you need to configure VSRX as well.</p>

<b> 5 Step </b>

<p>GRE configuration on the VSRX</p>
<p>You can choose how to configure VSRX via GUI or via ssh:</p>
<p>Via gui you can connect to private or public VSRX IP from GW configuration page:</p> 
<p>in our case link will be https://10.75.12.11:8443</p>
<p>To connect via Private IP you need to allow VPN access for your user and enable Motion Pro client VPN.</p> 
<p>Additional details here (https://cloud.ibm.com/docs/iaas-vpn?topic=iaas-vpn-standalone-vpn-clients#macos-standalone-client)</p>
<p>We will connect via ssh instead, with credentials available on the VSRX configuration page</p>
<p>First off all you need to create gre tunnel with following parameters:</p>
<p>Source IP 10.75.12.11 (VSRX private IP)</p>
<p>Destination 172.16.2.1 (GW of overlay subnet which you defined in virtual connection for Cloud Connection we defined 172.16.2.0/30 subnet)</p>
<p>GRE interface IP address on VSRX 172.16.2.6/30</p> ( You can assign yourselves available IP from this subnet )</p>
<p>Below necessary commands to create GRE tunnel on VSRX</p>

```shell
set interfaces gr-0/0/0 unit 0 tunnel source 10.75.12.11
set interfaces gr-0/0/0 unit 0 tunnel destination 172.16.2.1
set interfaces gr-0/0/0 unit 0 family inet mtu 1400
set interfaces gr-0/0/0 unit 0 family inet address 172.16.2.6/30
```
Respective configuration in other notation would be:
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

<p>next you need to allow security zones, because this traffic flow via private network we will allow all, but you can permit only explicit subnets etc if you need for security reason. We allow traffic between Power Zone which is belong to gre interface and allow to traverse traffic to IBM Cloud Classic which is SL-PRIVATE zone by default in IBM Cloud, also for troubleshooting allowed embedded zone junos-host to allow ping from loopback VSRX interfaces.</p>


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

In other notation it will looks following:

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
<p>The BGP neighbor the same IP which is used for GRE destination on the Power VS router end which is: 172.16.2.5</p>
<p>The Power VS ASN 64999</p>
<p>For VSRX ASN 64880</p>

```shell
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
We need to add policy to advertize prefixes from on-premise to Power VS router in on-premise we have two subnets

```shell
set policy-options policy-statement adver-prefix term 1 from route-filter 10.6.22.0/24 exact
set policy-options policy-statement adver-prefix term 1 from route-filter 10.5.11.0/24 exact
set policy-options policy-statement adver-prefix term 1 then accept
set policy-options policy-statement adver-prefix term default then reject
```
Then we need to allow BGP protocol in VSRX firewall for specific IP's most critical to allow it for overlay subnet 172.16.2.0/29  which is used for GRE tunnel 

```shell
set firewall filter PROTECT-IN term BGP from source-address 172.16.2.1/32
set firewall filter PROTECT-IN term BGP from source-address 169.254.0.1/32
set firewall filter PROTECT-IN term BGP from source-address 192.168.4.56/32
set firewall filter PROTECT-IN term BGP from source-address 172.16.2.5/32
set firewall filter PROTECT-IN term BGP from destination-address 169.254.0.1/32
set firewall filter PROTECT-IN term BGP from destination-address 172.16.2.1/32
set firewall filter PROTECT-IN term BGP from destination-address 169.254.0.2/32
set firewall filter PROTECT-IN term BGP from destination-address 10.75.12.11/32
set firewall filter PROTECT-IN term BGP from destination-address 172.16.2.6/32
set firewall filter PROTECT-IN term BGP from protocol tcp
set firewall filter PROTECT-IN term BGP from port bgp
set firewall filter PROTECT-IN term BGP then accept
```

We need to add static rotes for underlay network to proper routing traffice via Direct Link connection to VSRX transit VLAN 10.75.12.1 is GW for Private subnet attached to private VLAN

```shell
set routing-options static route 169.254.0.0/16 next-hop 10.75.12.1
set routing-options static route 172.16.2.1/32 next-hop 10.75.12.1
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

```
        route 169.254.0.0/16 next-hop 10.75.12.1;
        route 172.16.2.1/32 next-hop 10.75.12.1;
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
<p>If route advertized, you will then able to route this subnets to VPN tunnel later</p>
<p>To see which routes advertised from VSRX you can run followibg command </p> 

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
We see that both routes successfully advertized to PowerVS Router

<p>Regarding troubleshooting</p>
<p>You will not able to ping from PowerVS you VSRX private IP because it is underlay network for your GRE tunnel</p>
<p>anyway you can run ping on the Power VS instance and see how icmp flow to VPN interface when it will be configured later</p>
<p>You can do it with following command on the VSRX, to do so you need to provision at least one PowerVM in your PowerVS service instance, logon into this Vm via console and run ping some IP from the on-premise network range, for example 10.5.11.1</p> 

 ```shell
 run show security flow session protocol icmp
 ```

<b> 6 Step </b>
We need to establish VPN to on premise and route traffic from GRE to VPN interface st0.0 
<p>I will cover only configuration on the IBM Cloud side, on-premise configuration will be the same and only difference from specific device which used in the your organization.</p>

<p>the VSRX Public IP:  161.32.44.198</p>
<p>the on-premise VPN GW IP: 182.11.38.1</p>
<p>you need to replace pre-shared key "XXXXXXXXXXXXXXXX" to yours </p>
<p>If you prefer another dh-group and lifetime parameters you can change it as well</p>

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
Also we need to allow trafic between security zones from and to VPN

```shell
set security address-book global address net_10.6.22.0 10.6.22.0/24
set security address-book global address NET_10.75.12.0 10.75.12.0/26
set security address-book global address net_10.5.11.0 10.5.11.0/24
set security policies from-zone vpn to-zone SL-PRIVATE policy DR-to-private-net description "DR to privateNET"
set security policies from-zone vpn to-zone SL-PRIVATE policy DR-to-private-net match source-address net_10.6.22.0
set security policies from-zone vpn to-zone SL-PRIVATE policy DR-to-private-net match source-address net_10.5.11.0
set security policies from-zone vpn to-zone SL-PRIVATE policy DR-to-private-net match destination-address NET_10.75.12.0
set security policies from-zone vpn to-zone SL-PRIVATE policy DR-to-private-net match application any
set security policies from-zone vpn to-zone SL-PRIVATE policy DR-to-private-net match dynamic-application any
set security policies from-zone vpn to-zone SL-PRIVATE policy DR-to-private-net match url-category none
set security policies from-zone vpn to-zone SL-PRIVATE policy DR-to-private-net then permit
set security policies from-zone SL-PRIVATE to-zone vpn policy private-net-to-DR match source-address NET_10.75.12.0
set security policies from-zone SL-PRIVATE to-zone vpn policy private-net-to-DR match destination-address net_10.6.22.0
set security policies from-zone SL-PRIVATE to-zone vpn policy private-net-to-DR match destination-address net_10.5.11.0
set security policies from-zone SL-PRIVATE to-zone vpn policy private-net-to-DR match application any
set security policies from-zone SL-PRIVATE to-zone vpn policy private-net-to-DR match dynamic-application any
set security policies from-zone SL-PRIVATE to-zone vpn policy private-net-to-DR match url-category none
set security policies from-zone SL-PRIVATE to-zone vpn policy private-net-to-DR then permit
set security policies from-zone vpn to-zone POWER policy DR-to-PowerGRE match source-address any
set security policies from-zone vpn to-zone POWER policy DR-to-PowerGRE match destination-address any
set security policies from-zone vpn to-zone POWER policy DR-to-PowerGRE match application any
set security policies from-zone vpn to-zone POWER policy DR-to-PowerGRE then permit
set security policies from-zone POWER to-zone vpn policy PowerGRE-to-DR match source-address any
set security policies from-zone POWER to-zone vpn policy PowerGRE-to-DR match destination-address any
set security policies from-zone POWER to-zone vpn policy PowerGRE-to-DR match application any
set security policies from-zone POWER to-zone vpn policy PowerGRE-to-DR then permit
set security zones security-zone vpn description DR-NET
set security zones security-zone vpn interfaces st0.0
```
You need to allow ICMP for troubleshooting purposes, below configuration which I used in my case

```shell
set firewall filter PROTECT-IN term IKE from source-address 182.11.38.1/32
set firewall filter PROTECT-IN term IKE from destination-address 161.32.44.198/32
set firewall filter PROTECT-IN term IKE then accept
set firewall filter PROTECT-IN term ESP from source-address 182.11.38.1/32
set firewall filter PROTECT-IN term ESP from destination-address 161.32.44.198/32
set firewall filter PROTECT-IN term ESP from protocol esp
set firewall filter PROTECT-IN term ESP then accept
set firewall filter PROTECT-IN term PING from source-address 182.11.38.1/32
set firewall filter PROTECT-IN term PING from source-address 10.2.2.1/32
set firewall filter PROTECT-IN term PING from source-address 10.6.22.1/32
set firewall filter PROTECT-IN term PING from source-address 10.75.12.1/32
set firewall filter PROTECT-IN term PING from source-address 172.16.2.1/32
set firewall filter PROTECT-IN term PING from source-address 172.16.2.3/32
set firewall filter PROTECT-IN term PING from source-address 172.16.2.5/32
set firewall filter PROTECT-IN term PING from source-address 172.16.2.6/32
set firewall filter PROTECT-IN term PING from source-address 10.75.12.11/32
set firewall filter PROTECT-IN term PING from source-address 10.5.11.0/24
set firewall filter PROTECT-IN term PING from destination-address 161.32.44.198/32
set firewall filter PROTECT-IN term PING from destination-address 10.75.12.1/32
set firewall filter PROTECT-IN term PING from destination-address 10.75.12.11/32
set firewall filter PROTECT-IN term PING from destination-address 172.16.2.1/32
set firewall filter PROTECT-IN term PING from destination-address 172.16.2.3/32
set firewall filter PROTECT-IN term PING from destination-address 172.16.2.6/32
set firewall filter PROTECT-IN term PING from destination-address 172.16.2.5/32
set firewall filter PROTECT-IN term PING from destination-address 169.254.0.2/32
set firewall filter PROTECT-IN term PING from destination-address 169.254.0.1/32
set firewall filter PROTECT-IN term PING from destination-address 10.6.22.0/32
set firewall filter PROTECT-IN term PING from destination-address 195.66.81.1/32
set firewall filter PROTECT-IN term PING from destination-address 192.168.4.56/32
set firewall filter PROTECT-IN term PING from destination-address 10.12.248.2/32
set firewall filter PROTECT-IN term PING from destination-address 10.12.248.1/32
set firewall filter PROTECT-IN term PING from destination-address 10.2.2.1/32
set firewall filter PROTECT-IN term PING from destination-address 10.5.11.0/24
set firewall filter PROTECT-IN term PING from protocol icmp
set firewall filter PROTECT-IN term PING then accept
```
Finally we need to add route from PowerVS private network to VPN st0.0 interface

```shell
  set routing-options static route 10.5.11.0/24 next-hop st0.0
  set routing-options static route 10.6.22.0/24 next-hop st0.0
```

in other notation

```
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
            pre-shared-key ascii-text "$9$TzF/CtuBRh9Ct0B1EhcSrvM8XxdbYg8LUj"; ## SECRET-DATA
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

```
address-book {
        global {
            address net_10.6.22.0 10.6.22.0/24;
            address NET_10.75.12.0 10.75.12.0/26;
            address net_10.5.11.0 10.5.11.0/24;
             }
        }
```
```
}
        from-zone vpn to-zone SL-PRIVATE {
            policy DR-to-private-net {
                description "DR to privateNET";
                match {
                    source-address [ net_10.6.22.0 net_10.5.11.0 ];
                    destination-address NET_10.75.12.0;
                    application any;
                    dynamic-application any;
                    url-category none;
                }
                then {
                    permit;
                }
            }
        }
        from-zone SL-PRIVATE to-zone vpn {
            policy private-net-to-DR {
                match {
                    source-address NET_10.75.12.0;
                    destination-address net_10.6.22.0;
                    application any;
                    dynamic-application any;
                    url-category none;
                }
                then {
                    permit;
                }
            }
        }
 from-zone vpn to-zone POWER {
            policy DR-to-PowerGRE {
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
        from-zone POWER to-zone vpn {
            policy PowerGRE-to-DR {
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
        
         security-zone vpn {
            description DR-NET;
            interfaces {
                st0.0;
            }
        }
```

```
filter PROTECT-IN {
        term IKE {
            from {
                source-address {
                    182.11.38.11/32;
                }
                destination-address {
                    161.32.44.198/32;
                }
            }
            then accept;
        }

        term PING {
            from {
                source-address {
                    182.11.38.11/32;
                    10.2.2.1/32;
                    10.6.22.1/32;
                    10.75.12.1/32;
                    172.16.2.1/32;
                    172.16.2.3/32;
                    172.16.2.5/32;
                    172.16.2.6/32;
                    10.75.12.11/32;
                    10.5.11.0/24;
                }
                destination-address {
                    161.32.44.198/32;
                    10.75.12.1/32;
                    10.75.12.11/32;
                    172.16.2.1/32;
                    172.16.2.3/32;
                    172.16.2.6/32;
                    172.16.2.5/32;
                    169.254.0.2/32;
                    169.254.0.1/32;
                    10.6.22.131/32;
                    195.66.81.1/32;
                    192.168.4.56/32;
                    10.12.248.2/32;
                    10.12.248.1/32;
                    10.2.2.1/32;
                }
                protocol icmp;
            }
            then accept;
        }
        term ESP {
            from {
                source-address {
                    182.11.38.11/32;
                }
                destination-address {
                    161.32.44.198/32;
                }
                protocol esp;
            }
            then accept;
        }
```

```
   static {
        route 10.6.22.0/24 next-hop st0.0;
        route 10.5.11.0/24 next-hop st0.0;
    }
```
<p>Then you need to repeat necessary steps on the on-premise VPN GW device.</p>
<p>If you have issue you can find some examples in the IBM Cloud Documentation here: https://cloud.ibm.com/docs/vpc?topic=vpc-cisco-asav-config&interface=ui </p>

<p> You can find here end to end configuration for the scenario described above:</p>

https://github.com/notras/PowerVSConnectivity/blob/main/gateway-vsrx-vSRXfullexample.conf


<b>Additional information:</b>
<p>Very useful guide from my coleague Chris. This scenario use TGW in advance and Direct link for connect to on-premise infrastructure:</p>

https://cloudguy.ca/2022/03/19/connecting-to-ibm-power-systems-virtual-servers-through-direct-link/

<p>IBM cloud documentation:</p>
<p>PowerVS:</p>
https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-network-architecture-diagrams#network-reference-architecture-pvs2pvs
https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-cloud-connections#configure-gre-tunnel
<p>JuniperVSRX configuration:</p>
https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-cloud-connections#configure-gre-tunnel
