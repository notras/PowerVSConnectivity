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
 1. VPN GW in VPC + TGW to PowerVS cloud connection 
 2. GW appliance + Power VS cloud connection
 
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

<b>1 step</b>
Create Power VS service insrtance
https://cloud.ibm.com/catalog/services/power-systems-virtual-server

![Creating PowerVS instance](https://github.com/notras/PowerVSConnectivity/blob/main/powerVSinstanceceration.png)

<b>2 step </b>
Create subnets inside PowerVS service instance, the main reason, it should exist before you order Direct Link 2.0 connection from PowerVS colo to the rest IBM cloud resources, the main reason proper BGP routing.
![Creating Private Network](https://github.com/notras/PowerVSConnectivity/blob/main/PrivateNWcreation.png)
<b> 3 step </b>
We need to order direct link which will connect PowerVS with IBM Cloud services and attach previously created subnet 10.2.2.0/24 or several subnets to the Cloud Connection. IBM provide free of charge option for interconnection between PowerVS Colo and rest of the IBM Cloud services.
![Creating Direct Link PowerVS to IBM Cloud](https://github.com/notras/PowerVSConnectivity/blob/main/DirectLinkPower.png)

useful link https://cloudguy.ca/2022/03/19/connecting-to-ibm-power-systems-virtual-servers-through-direct-link/
