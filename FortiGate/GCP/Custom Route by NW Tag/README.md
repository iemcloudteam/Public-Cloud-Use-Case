# FortiGate HA - Egress Inspection with custom-route using network tag in GCP
This document describes how to inspect outgouing traffic to Internet through FortiGate with multi-zone HA cluster setup using network-tag feature set per Virtual Machine (VM) in Google Cloud Platform (GCP). Main idea is to shift outgoing traffic path to FortiGate instances individually per Spoke-VM in step-by-step fashion.

Official document for network-tags can be found at [here](https://cloud.google.com/vpc/docs/add-remove-network-tags)

In this scenario, FortiGate active/passive (A/P) cluster is operating as Internet breakout firewall in GCP project. "Custom route" is a GCP networking component that we can use to manipulate default routing behavior for inter-VPC type of traffic. In normal conditions, each subnet within a VPC uses default route automatically assigned by platform pointing GCP's Internet services. Egress traffic to Internet can be routed through following next-hop options:

-	Instance (available in GUI)
-	IP address (available in GUI)
-	VPN tunnel (available in GUI)
-	Forwarding rule of internal TCP/UDP load balancer (available in GUI)
-	Internal TCP/UDP load balancer IP address (**not-available** in GUI)

Last next-hop option cannot be configured using GCP GUI as of today (Mar'22). Configuration will be done using gcloud CLI commands which is described below.

## Pre-requisites
-	Fortigate multi-zone HA A/P deployed in project using [deployment templates](https://github.com/40net-cloud/fortinet-gcp-solutions/tree/master/FortiGate/architectures/200-ha-active-passive-lb-sandwich),
-	Spoke VPCs and VMs deployed in each VPC,
-	VPC peering established between Spoke VPCs and FortiGate Internal VPC ([configuring VPC peering](https://cloud.google.com/vpc/docs/vpc-peering))
-	FortiGate NAT-enabled security rule allowing specific egress services for Spoke-VMs
-	FortiGate static route for Spoke-VPC CIDRs
-	*(Optional)* FortiGate Fabric-connector to import GCP objects ([FortiGate GCP Fabric-connector how to](https://docs.fortinet.com/document/fortigate-public-cloud/6.2.0/gcp-administration-guide/906632/security-fabric-connector-integration-with-gcp))

## Design & Topology
The following diagram illustrates environment for this use-case. As shown in the topology, there is Internal Load Balancer (ILB) placed behind FortiGate HA cluster. ILB's internal fronting IP address will be used as a next-hop-IP-address setting in custom-route configuration pointing out to Internet. 

<img src=https://github.com/iemcloudteam/azure_paas_services_inspection/blob/f7bc1a1e5a3fc5e05e9e64718eefb21bdc44d654/images/topology.png width="1000"/>

## Configuration Steps
When you are configuring Private Endpoint there is a routing entry that will be injected into your routing table that look as follow.

<img src=https://github.com/iemcloudteam/azure_paas_services_inspection/blob/8aa82bcd6ce72c1d66d9d851fdd277cff6d1c456/images/routing.png width="800"/>

So, all the traffic targeted to the Private Endpoint interface will be routed through the next hop type which is “InterfaceEndpoint”. Since our task is to push all that traffic through the Fortigate, we need to introduce additional routing entry that looks as follows:

<img src=https://github.com/iemcloudteam/azure_paas_services_inspection/blob/8aa82bcd6ce72c1d66d9d851fdd277cff6d1c456/images/routing2.png width="800"/>

## Using Fortigate as DNS Forwarder.
1.	Under "System > Feature Visibility", you need to enable "DNS Database" feature.
2.	Go to "Network > DNS Servers", under "DNS Service on Interface" we need to run DNS proxy on the specific network interface. 

    Because my task was to use FortiGate as DNS Forwarder for my “ProtectedB” subnet and for my VPN tunnel I had to add two following entries:
    
    <img src=https://github.com/iemcloudteam/azure_paas_services_inspection/blob/8aa82bcd6ce72c1d66d9d851fdd277cff6d1c456/images/DNS1.png width="400"/>

    First one is for “ProtectedB” subnet that is connected to FortiGate via Port2. We're forwarding DNS queries to System DNS.
    
    <img src=https://github.com/iemcloudteam/azure_paas_services_inspection/blob/8aa82bcd6ce72c1d66d9d851fdd277cff6d1c456/images/DNS2.png width="400"/>

    The second one is for IPSECVPN interface

    <img src=https://github.com/iemcloudteam/azure_paas_services_inspection/blob/8aa82bcd6ce72c1d66d9d851fdd277cff6d1c456/images/DNS3.png width="400"/>

3.	 Finally, because we want to forward DNS queries to the System DNS we need to check "Network > DNS configuration". By default, FortiGate will be using FortiGuard DNS servers. We need to change it to internal Azure DNS server which is listening on 168.63.129.16.
	
    <img src=https://github.com/iemcloudteam/azure_paas_services_inspection/blob/8aa82bcd6ce72c1d66d9d851fdd277cff6d1c456/images/DNS4.png width="400"/>
    
 More information about using FortiGate as a DNS Proxy can be foud [here](https://docs.fortinet.com/document/fortigate/6.2.10/cookbook/121810/using-a-fortigate-as-a-dns-server)
