# SDN_Path_Checker_DNS_DHCP
SDN DNS_Blacklist, Path_check Application
The lab involves working with hybrid switches and vendor-neutral servers in setting
up the Software-Defined Network that controls and automates the network configuration,
monitoring and the management. The objective involves setting up and installing an SDN
controller on an x86 platform, configuring Open vSwitch on bare metal boxes, understanding the
CLI of prominent vendors that support OpenFlow, basic routing, switching, and network
engineering design and troubleshooting OpenFlow and network design and programming
applications via API and OpenFlow

Topology :

<img width="650" alt="image" src="https://github.com/feralforreal/SDN_Path_Checker_DNS_DHCP/assets/132085748/a94ddae8-32ae-4941-bebf-4535c44e7037">


Physical Hardware Spec:
- OvS Dell AMBX Server, OvS HP server, and OvS Dell Server
- RYU Controller
- Hybrid Aritsa, HP switches
- Cisco Router working as a Internet Router
- Cisco L2 switch working as a Management Switch

I. Automated OpenFlow configuration
1. Create an application that automates the OpenFlow configuration to the controller for
all the hardware switches.
2. The application should fetch all the required connection information (switch IP,
controller IP, OpenFlow port, OpenFlow version, OpenFlow transport protocol, etc.)
from a NSOT file and then configure the switches accordingly.
3. The application should verify and display that each switch has established successful
OpenFlow connectivity with the controller.

<img width="616" alt="image" src="https://github.com/feralforreal/SDN_Path_Checker_DNS_DHCP/assets/132085748/e54deff7-c0ee-4079-9b2b-d6d91308ec93">

DNS_Blacklist - 
Applications functionality explained :
DNS Blacklist - The application is used to prevent certain IP addresses or domains from
communicating with a network or accessing services based on their inclusion in a blacklist.
These lists contain the IP addresses or domain names of servers or sources that have been
identified as sending spam, hosting malware, or engaging in other malicious activities. In
our case, we are allowing the google, youtube and Colorado as allowed sites.

And the algorithm works as :
If packet_in (dstn_port = 53 && ip.protocol = udp ):
1. Parse the DNS packet and fetch the URL:
a. Check if the URL is locally whitelisted:
[need to do this to reduce the load on flask application serving as site checker]
i. If allowed locally:
1. Generate a POST request to check the site on remote flask
application.
2. If allowed, then mark the site as ALLOWED
ii. Else:
1. Mark the site as NOT ALLOWED

b. If ALLOWED:
i. Install a reverse flow for the DNS REPLY.
ii. Send the packet to the next hop towards the gateway.
c. Else:
i. Craft a DNS REPLY using scapy.(DNS Resolution = controllerIP)
[since the browser will generate a TCP request with
http://controllerIP:443 it will redirect it to a page showing the site is
blocked]
ii. Send the packet crafted on to the in_port.

<img width="458" alt="image" src="https://github.com/feralforreal/SDN_Path_Checker_DNS_DHCP/assets/132085748/6a2b98e5-cef7-4e03-91ba-24b7d4e009da">


<img width="613" alt="image" src="https://github.com/feralforreal/SDN_Path_Checker_DNS_DHCP/assets/132085748/21ae7cd3-4b11-4f41-9f1e-3c20b4b89a99">


<img width="818" alt="image" src="https://github.com/feralforreal/SDN_Path_Checker_DNS_DHCP/assets/132085748/6a2fe380-d23e-4d90-9305-c07a8c94f71f">

<img width="818" alt="image" src="https://github.com/feralforreal/SDN_Path_Checker_DNS_DHCP/assets/132085748/dd9ea7d3-1d2f-4064-b59d-371f36697dc1">

<img width="517" alt="image" src="https://github.com/feralforreal/SDN_Path_Checker_DNS_DHCP/assets/132085748/8fc29e39-d849-411c-b607-200cc3f8d286">







DHCP Application: 
It's a network protocol that's responsible for automatically assigning IP
addresses and other network configuration information to devices within a network. DHCP
assigns IP addresses dynamically to devices on a network, eliminating the need for manual
configuration. When a device connects to a network, the DHCP server provides it with an IP
address from a pool of available addresses. Besides IP addresses, DHCP can also provide
additional network configuration details, such as subnet mask, default gateway, DNS server
information, and more. This ensures that devices connecting to the network have all the
necessary settings to communicate properly. In our case, we are configuring DHCP using app.
And the algorithm works as:

If packet_in (dst_port = 67 && ip.protocol =udp && dst_ip = 255.255.255.255):
1. Extract all the ports on the switch towards DHCP server.
2. Check if these extracted ports are UP.
3. Install flows for DHCP DISCOVER towards the DHCP server, with all the available egress
ports, with decremental priorities.
The priority with shortest path is greater, then the next greater and so on.
4. Install flow for DHCP reply from the DHCP server:
[We install flows with egress ports only if the port is UP]
a. Check all the ports on the switch towards the host.
b. Extract the ports and install flows with decremental priority.
5. Instruct the switch to send the port out on the port with highest priority.

<img width="610" alt="image" src="https://github.com/feralforreal/SDN_Path_Checker_DNS_DHCP/assets/132085748/ab3f6d05-dd05-49fa-9900-240434866a5e">


Path Tracer:
Path tracer application work with ICMP. When the edge device generates a PING request the
controller received the ECHO message and starts tracking the ECHO request’s in_port and
out_port. Once, the Echo Packet is thrown towards the Gateway, the controller sends a POST
request of the final ICMP path formed, to the flask Application on endpoint /updatetopoPath.
The flask application continuously refreshes the page and prints the ICMP path.
Alogrithm is as follows:
global ICMP_PATH
if packet_in( ip.protocol = ICMP && destination_MAC = GatewayMAC):
1. Extract all the ports on the switch towards the DHCP server.
2. Check if the extracted ports are UP.
3. If the ECHO request received from the edge Port (port towards the host):
a. Record the (inPort, outport, switch-Openflow-Channel IP) in ICMP_PATH
4. If the ECHO request received from a non-edge port AND

Outport is not the Gateway Port:
a. Record the (inPort, outport, switch-Openflow-Channel IP) in ICMP_PATH
5. If the outport of ECHO Request is a Gateway Port:
a. Record the (inPort, outport, switch-Openflow-Channel IP) in ICMP_PATH
b. Send the POST request to flask application to update ICMP_PATH


Failover Tracking:
While installing flows we check if the port is UP before installing the flow, by doing a local lookup
into the local state table of all ports on all switches.
Anytime a wire is plugged out, a PortStats event is sent asynchronously from the switch
to the controller, we detect that and update the local state table of ports. Additionally, we
remove flows for that port the switch, with that egress port. This way, the controller can install
new flows since the switch won’t have flows. Thus, new flows are added with new ports that are
UP.


<img width="511" alt="image" src="https://github.com/feralforreal/SDN_Path_Checker_DNS_DHCP/assets/132085748/a09d504f-64c0-465e-929d-775485aa99e0">





