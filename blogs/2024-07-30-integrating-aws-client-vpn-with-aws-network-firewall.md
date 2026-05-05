---
title: "Integrating AWS Client VPN with AWS Network Firewall"
url: "https://aws.amazon.com/blogs/networking-and-content-delivery/integrating-aws-client-vpn-with-aws-network-firewall/"
date: "Tue, 30 Jul 2024 18:20:42 +0000"
author: "Devansh Agrawal"
feed_url: "https://aws.amazon.com/blogs/networking-and-content-delivery/tag/aws-network-firewall/feed/"
---
<p>Organizations use remote access solutions for secure remote user access to resources hosted on their internal networks. This post shows various deployment models to integrate <a href="https://aws.amazon.com/network-firewall/">AWS Network Firewall</a> with <a href="https://aws.amazon.com/vpn/client-vpn/">AWS Client VPN</a>. AWS Client VPN is a managed client-based VPN service that secures access to your AWS resources, and resources in your on-premises network, over <a href="https://aws.amazon.com/vpn/site-to-site-vpn/">AWS Site-to-Site VPN</a> or <a href="https://aws.amazon.com/directconnect/">AWS Direct Connect</a>. With Client VPN, you can access your resources from any location using an OpenVPN-based VPN client</p> 
<p>You can inspect remote access VPN traffic using AWS Network Firewall to perform:</p> 
<ul> 
 <li>Fine-grained control over network traffic. For example, blocking outbound Server Message Block (SMB) requests to prevent the spread of malicious activity.</li> 
 <li>Web filtering that can stop traffic to known bad URLs and monitor fully qualified domain names.</li> 
 <li>Intrusion prevention (IPS)/Intrusion detection (IDS) which provides active traffic flow inspection so you can identify and block vulnerability exploits using signature-based detection.</li> 
</ul> 
<p>Integrating Network Firewall with Client VPN allows you to integrate network security for your remote users’ traffic. You can control access to your workloads and protect against malicious traffic originating from remote users. Network Firewall is a managed service that makes it easy to deploy essential network protections for all of your <a href="https://aws.amazon.com/vpc/">Amazon Virtual Private Clouds (Amazon VPCs)</a> and workloads.</p> 
<h2>How AWS Client VPN routing works</h2> 
<p>At present, the Client VPN provides access to both AWS resources and non-AWS resources through two types of routing modes:</p> 
<ol> 
 <li><strong>Tunnel All</strong> — By default, when you have a Client VPN endpoint, all traffic from clients is routed over the Client VPN tunnel after you are connected to the endpoint. This not only includes your VPC traffic but also internet traffic, such as example.com.</li> 
 <li><strong>Split Tunnel</strong> — You can use Client VPN endpoint in split-tunnel mode when you do not want all user traffic to route through the Client VPN endpoint. When you enable split-tunnel, your Client VPN Endpoint pushes the routes from the route table configured at <a href="https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/cvpn-working-routes.html">Client VPN endpoint</a>. This ensures that traffic matching a route from the endpoint route table is routed over the Client VPN tunnel, while other traffic is routed through the local internet provider.</li> 
</ol> 
<h2>Deployment patterns</h2> 
<p>You can deploy Network Firewall with Client VPN in various patterns. This section describes the different deployment patterns with benefits and considerations for each architecture. The right model depends on the use case and requirements. It depends also on how you want to extend Client VPN connectivity to your existing environment. Let’s look at two deployment patterns:</p> 
<ol> 
 <li>Single VPC architecture with Client VPN and Network Firewall integration</li> 
 <li>Network Firewall in a centralized model for Client VPN Traffic</li> 
</ol> 
<h3><strong>Single VPC architecture with Client VPN and Network Firewall integration</strong></h3> 
<p>Considering that you have your workload running within a single VPC, and you want your remote users to access the resources in that VPC. In this scenario, you can create separate Network Firewall and Client VPN subnets in the existing VPC, and create Network Firewall endpoints in firewall subnets. Furthermore, you can create a Client VPN endpoint by going to the <a href="https://console.aws.amazon.com/vpc/">VPC console</a> and attaching Client VPN subnets as the target network. This creates Client VPN Elastic Network Interfaces (ENI) in the Client VPN Subnets.</p> 
<p>By design, all forward traffic from Client VPN users gets Source NATed to Client VPN interface IPs. Hence, return traffic from workloads or Targets is always destined for Client VPN network interface IPs. For return traffic, you must use the <a href="https://aws.amazon.com/blogs/aws/inspect-subnet-to-subnet-traffic-with-amazon-vpc-more-specific-routing/">more specific routing</a> feature in the workload subnet route table. You must add routes for the destinations as Client VPN subnets ranges and Network Firewall endpoint as the Targets.</p> 
<p>Each Client VPN subnet requires a dedicated VPC route table to make sure you are forwarding the traffic to a firewall endpoint within the same <a href="https://aws.amazon.com/about-aws/global-infrastructure/regions_az/">Availability Zone (AZ)</a>. This avoids inter-AZ data transfer charges and traffic asymmetry. Network Firewall endpoints are zonal in nature. Therefore, the Network Firewall endpoint that processed the forward traffic must also process the return traffic. Figure 1, shows an architecture with Client VPN Subnets and Network Firewall endpoints in two AZs.</p> 
<div class="wp-caption aligncenter" id="attachment_23889" style="width: 1124px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/09/11/Client-VPN-Blog-1.png" rel="noopener" target="_blank"><img alt="Client VPN and Network Firewall integraton for single VPC" class="wp-image-23889 size-full" height="689" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/09/11/Client-VPN-Blog-1.png" width="1114" /></a>
 <p class="wp-caption-text" id="caption-attachment-23889">Figure 1: Client VPN and Network Firewall integration (click to open a larger version)</p>
</div> 
<p>Now, let’s look at the traffic flow called out with numbers in figure 1. This flow occurs when a remote user initiates the connection request.</p> 
<ol> 
 <li>Remote users are connected to a Client VPN Endpoint using the VPN Client.</li> 
 <li>The users’ traffic lands on Client VPN Subnets present in the workload VPC and gets Source NATed to the Client VPN Elastic Network Interface IP address.</li> 
 <li>The route for VPC CIDR Range and the default route in the Client VPN subnet route table are configured to forward the traffic to firewall endpoints. Therefore, based on where the subnet and AZ traffic lands, it is forwarded to the respective firewall endpoint.</li> 
 <li>Depending on the destination, traffic is forwarded differently based on the routes configured in the firewall Subnet Route Tables.Traffic destined to the internet is routed through <a href="https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html">NAT Gateway</a>.</li> 
 <li>Traffic to workloads present in same VPC goes directly from Network Firewall endpoints post inspection.</li> 
 <li>The NAT Gateway subnet has a route toward an <a href="https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html">Internet Gateway (IGW)</a>. Source IP translation happens at the NAT Gateway, and traffic is routed to an Internet Gateway (IGW).</li> 
 <li>The IGW forwards internet-bound traffic based on the destination that the Client VPN user is trying to connect to.</li> 
</ol> 
<h4>Benefits and considerations</h4> 
<p>This architecture has the following benefits and considerations:</p> 
<ul> 
 <li>It is simple to set up. With no major changes to your existing VPC Route Tables, Network Firewall can be inserted transparently between Client VPN users and workload traffic.</li> 
 <li>Using the more specific routing feature (MSR feature of VPC), Client VPN users’ traffic is inspected first by Network Firewall before forwarding it to workloads. Refer to Client VPN Subnet RT in the preceding figure.</li> 
 <li>There is no additional cost for <a href="https://aws.amazon.com/network-firewall/pricing/">NAT Gateways when Network Firewall</a> is running, provided both are part of the same account.</li> 
 <li>If you intend to direct Client VPN traffic towards the internet, the Client VPN Elastic Network Interfaces (ENIs) are assigned public IP addresses by default. Hence, you do not need NAT Gateways for private to public IP translation. However, these public IPs can change. If you must have a fixed static public IP address, you can forward remote users’ traffic through NAT Gateway, which provides fixed Elastic IP Address. Refer to the preceding figure, where NAT Gateways are used to provide the static Elastic IP address for the Client VPN ENIs when traffic exits towards the internet.</li> 
 <li>If remote users belong to different active directory groups and have specific network access requirement by individual groups, you can use the <a href="https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/cvpn-working-rules.html">authorization rules of Client VPN</a> to specify network access based on users group. Network Firewall rules are created based on the 5 tuple (Source and Destination IP and Port, Protocol) or domain, so they cannot restrict the traffic based on the active directory groups.</li> 
</ul> 
<h3>Network Firewall in centralized model for Client VPN Traffic</h3> 
<p><a href="https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html">AWS Transit Gateway</a> is a prerequisite of centralized deployment patterns. It acts as a network hub and simplifies the connectivity between VPCs and on-premises networks. In the centralized model, one VPC is dedicated as a centralized inspection VPC for north-south and east-west traffic, where you deploy Network Firewall and centralized NAT Gateway for Egress Internet Traffic.</p> 
<p>In this scenario, to extend the connectivity of your workloads to remote users using AWS Client VPN, you use an inspection VPC as the Target VPC for Client VPN. The routing configurations are similar to the first model. This pattern extends the first one, but adds more connectivity to other workload VPCs using Transit Gateway. Figure 2 shows this architecture.</p> 
<div class="wp-caption aligncenter" id="attachment_23890" style="width: 1597px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/09/11/Client-VPN-Blog-2.png" rel="noopener" target="_blank"><img alt="Client VPN and Network firewall for multiple VPC's using transit gateway" class="wp-image-23890 size-full" height="816" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/09/11/Client-VPN-Blog-2.png" width="1587" /></a>
 <p class="wp-caption-text" id="caption-attachment-23890">Figure 2: Centralized model – AWS Client VPN and Network Firewall in the same VPC (click to open a larger version)</p>
</div> 
<p>You might need to land Client VPN traffic in a different VPC from the inspection VPC because:</p> 
<ul> 
 <li>You want to grant some users temporary access to applications. Here, using the same VPC for Client VPN and inspection does not give you fine-grained routing controls at the Transit Gateway, as the same route table is shared between Client VPN and the Inspection VPC. (Refer to the preceding figure for Transit Gateway route table details of the inspection VPC.) Therefore, Client VPN users can get access to other unintended workload VPCs.</li> 
 <li>In the preceding scenario, you may also want to segment traffic from each remote user with temporary access. In this case, access through different Client VPN Endpoints and Target VPCs gives you more security controls. You can enforce fine-grained routing controls on Transit Gateway by having different route tables for individual Client VPN Target VPCs.</li> 
</ul> 
<p>For these scenarios, create separate Target VPCs for Client VPN Endpoints and connect them to specific workloads using Transit Gateway. Then create a separate routing table on the transit gateway for the Client VPN Target VPC. You can use your existing Inspection VPC and Network Firewall endpoints for inspecting Client VPN traffic. This gives you better controls over routing Client VPN traffic.</p> 
<p>Refer to the Client VPN Target VPC and Transit Gateway route tables in figure 3 for routing rules</p> 
<div class="wp-caption aligncenter" id="attachment_23891" style="width: 1471px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/09/11/Client-VPN-Blog-3.png" rel="noopener" target="_blank"><img alt="Centralized model, AWS Client VPN and Network Firewall in different VPCs " class="wp-image-23891 size-full" height="1386" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/09/11/Client-VPN-Blog-3.png" width="1461" /></a>
 <p class="wp-caption-text" id="caption-attachment-23891">Figure 3: Centralized model – AWS Client VPN and Network Firewall in different VPC (click to open a larger version)</p>
</div> 
<p>As shown in the preceding figure, Client VPN Subnets and Network Firewall endpoints are part of different VPCs and are connected through Transit Gateway. Let’s see the traffic flow for a remote user accessing the resources of UAT VPC.</p> 
<ol> 
 <li>Remote users are connected to Client VPN Endpoint using VPN Client.</li> 
 <li>User traffic lands on Client VPN Subnets inside the Client VPN VPC and gets Source NATed.</li> 
 <li>The route for the UAT VPC CIDR Range in the Client VPN subnet route table is configured to forward the traffic to Transit Gateway. Therefore, based on where the subnet and AZ traffic lands, it is forwarded to the respective Transit Gateway Subnet ENIs.</li> 
 <li>On referring to Client VPN VPC Transit Gateway Route table, traffic is forwarded to the inspection VPC attachment.</li> 
 <li>Traffic will land on to any of the attached Transit Gateway subnets and AZs in the inspection VPC.</li> 
 <li>Based on where the subnet and AZ traffic lands in the inspection VPC, it is forwarded to respective Firewall Endpoints and inspected.</li> 
 <li>Referring to the Network Firewall Subnet Route table, traffic is forwarded back to Transit Gateway post inspection.</li> 
 <li>Traffic refers to the Inspection VPC Transit Gateway Route table and is forwarded to the Transit Gateway ENI present in the UAT VPC.</li> 
 <li>From there, following the local route of the VPC, traffic is routed to the workload instance.</li> 
</ol> 
<p>Note that the return traffic follows the same path in the opposite direction.</p> 
<h4>Let’s see the traffic flow for a EC2, accessing the resources on Internet;</h4> 
<p>(A) Traffic from EC2 is routed to TGW referring to Subnet Route Table</p> 
<p>(B) On referring to Spoke VPC Transit Gateway Route table, traffic is forwarded to the inspection VPC attachment. Traffic will land on to any of the attached Transit Gateway subnets and AZs in the inspection VPC.</p> 
<p>(C) Based on where the subnet and AZ traffic lands in the inspection VPC, it is forwarded to respective Firewall Endpoints and inspected.</p> 
<p>(D) Referring to the Network Firewall Subnet Route table, traffic is forwarded to NAT Gateway post inspection.</p> 
<p>(E) From there, following the NAT Gateway Subnet route table of the Inspection VPC, traffic is routed to the Internet through Internet Gateway.</p> 
<p>Note that the return traffic follows the same path in the opposite direction.</p> 
<p>Benefits and considerations, Consider the following when planning your project:</p> 
<ul> 
 <li>Using the same Network Firewall for Client VPN traffic avoids additional hourly charges that occur when you have separate firewall endpoints for Client VPN traffic. You can share the existing firewall endpoints across multiple Client VPN Target VPCs.</li> 
 <li>The Client VPN Target VPC acts as an attachment to Transit Gateway. Therefore, you will be charged for one Transit Gateway attachment (as of when this post was published). However, you gain additional routing controls by having a separate routing table on Transit Gateway for Client VPN Traffic, apart from allowing or denying the traffic at Network Firewall.</li> 
 <li>There is no additional cost of NAT Gateway for the amount of hours Network Firewall is running.</li> 
 <li>Allow and Deny access policies for different environments, such as Production and UAT, must be created on Network Firewall. Keep in mind that the source is always a Client VPN network interface IP address.</li> 
 <li>For third-party remote users access, only give access to a specific network range using AWS <a href="https://aws.amazon.com/network-firewall/pricing/">Client VPN authorization rules</a>.</li> 
</ul> 
<p>You can also deploy Network Firewall for Client VPN traffic in the Client VPN Target VPC. This is valuable if you must have individual firewall endpoints. This gives you the flexibility to create a separate set of firewall policies for remote user traffic and inspect it before Transit Gateway. However, (as of when this post published) there are additional hourly charges for separate firewall endpoints for Client VPN traffic.</p> 
<p>When integrating AWS Client VPN with Network Firewall, remember that:</p> 
<ul> 
 <li>Although Network Firewall is represented by a single firewall endpoint per subnet/AZ, it is highly available and based on <a href="https://youtu.be/8gc2DgBqo9U?t=2061">AWS Hyperplane technology</a>. No single component failure of Network Firewall would cause an AZ failure.</li> 
 <li>Firewall endpoints are deployed in all AZs used by your workloads so that traffic is inspected within the same AZ to avoid inter AZ data transfer charges.</li> 
 <li>Transit Gateway has <a href="https://aws.amazon.com/transit-gateway/pricing/">data processing and attachment costs</a>. Transit Gateway is required for centralized models. To avoid Transit Gateway data processing costs caused by unwanted traffic from Client VPN, traffic must be blocked at the AWS Network Firewall and by specific Client VPN authorization rules before it is routed to Transit Gateway for further routing and processing.</li> 
 <li>Make sure that <a href="https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-appliance-scenario.html">appliance mode</a> is enabled on the inspection VPC attachment. This makes sure that bidirectional traffic forwarding to the same Firewall Endpoint.</li> 
 <li>Network Firewall cannot inspect traffic between VPCs that are peered together (as of when this post was published). This is also true for on-premises networks that are connected directly to a VPC using Virtual Private Gateway.</li> 
 <li>AWS Client VPN traffic is source NATed to a network interface IP address in the Client VPN Target VPC. Therefore, access policy at Network Firewall cannot be created based on the Client CIDR IPs/Range.</li> 
</ul> 
<h3>Conclusion</h3> 
<p>AWS Network Firewall transparently inspects traffic at scale in a variety of use cases. The right deployment model depends on the desired outcome. There are a handful of private entry points for applications hosted inside Amazon VPCs. You can access your application over private IPs using VPC Peering, VPN, AWS Direct Connect, AWS Private Link, and AWS Client VPN. Understanding all the private entry points in AWS and determining the right strategy for deep packet inspection helps strengthen the overall security posture of your infrastructure. Integrating AWS Client VPN with AWS Network Firewall provides additional protection when your applications are accessed by trusted end users. By deploying AWS Network Firewall with AWS Client VPN using the deployment patterns described in this post, you can seamlessly integrate network security for your remote users’ traffic.</p> 
<h3>Further reading:</h3> 
<p>·&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <a href="https://aws.amazon.com/blogs/aws/inspect-subnet-to-subnet-traffic-with-amazon-vpc-more-specific-routing/">Amazon VPC Routing Enhancements with Most Specific Routing feature</a></p> 
<p>·&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <a href="https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/">AWS Network Firewall Deployment patterns</a></p> 
<p>·&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <a href="https://aws.amazon.com/blogs/networking-and-content-delivery/scale-remote-access-vpn-on-aws/">Scale your Remote Access VPN on AWS</a></p> 
<h3><strong>About the author</strong></h3> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="size-full wp-image-11636 alignleft" height="160" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/06/22/devansh-3.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Devansh Agrawal</h3> 
  <p>Devansh is a Sr. Cloud Support Engineer at AWS. He is passionate about assisting customers in enhancing their systems to protect from DDOS attacks and Layer 7 attacks.Devansh enjoys working with customers to solve their technical challenges and create secure and scalable architectures on the AWS Cloud.Outside of work, Devansh loves trying different cuisines and traveling. LinkedIn: /devansh-agrawal-a1513614b/</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="size-full wp-image-11636 alignleft" height="160" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/12/26/Avanish-Blog-1.png" width="120" />
  </div> 
  <h3 class="lb-h4">Avanish Yadav</h3> 
  <p>Avanish Yadav is a Senior Networking Solutions Architect at Amazon Web Services. With a passion for networking technologies, he enjoys innovating and helping customers solve complex technical challenges by creating secure, scalable cloud architectures. When he’s not collaborating with clients to provide expert solutions to their needs, he can often be found playing cricket outside of work. LinkedIn: /avanish-yadav-93b8a947/</p> 
 </div> 
</footer>
