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
  
![PowerVS-to-on-Premise-Architecture](https://github.com/notras/PowerVSConnectivity/blob/main/GREIpsecPowerVS-GRE.drawioV1.png)

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
Public IP 161.32.44.122

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
edit
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
```

Very useful guide from my coleague: https://cloudguy.ca/2022/03/19/connecting-to-ibm-power-systems-virtual-servers-through-direct-link/

IBM cloud documentation:
PowerVS:
https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-network-architecture-diagrams#network-reference-architecture-pvs2pvs
https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-cloud-connections#configure-gre-tunnel
JuniperVSR configuration:
https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-cloud-connections#configure-gre-tunnel
