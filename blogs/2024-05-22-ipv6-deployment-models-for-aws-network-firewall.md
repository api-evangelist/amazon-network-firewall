---
title: "IPv6 deployment models for AWS Network Firewall"
url: "https://aws.amazon.com/blogs/networking-and-content-delivery/ipv6-deployment-models-for-aws-network-firewall/"
date: "Wed, 22 May 2024 16:40:59 +0000"
author: "Tushar Jagdale"
feed_url: "https://aws.amazon.com/blogs/networking-and-content-delivery/tag/aws-network-firewall/feed/"
---
<p><a href="https://aws.amazon.com/network-firewall/">AWS Network Firewall</a> is a managed, stateful network firewall and intrusion protection service that allows you to implement firewalls rules for fine grained control over your network traffic. If you’re new to AWS Network Firewall, and want to understand its <a href="https://aws.amazon.com/network-firewall/features/">features</a> and use cases, we recommend you review the blog post <a href="https://aws.amazon.com/blogs/aws/aws-network-firewall-new-managed-firewall-service-in-vpc/">AWS Network Firewall – New Managed Firewall Service in VPC</a>.</p> 
<p>In this post, we show how you can use AWS Network Firewall in dual stack and IPv6-only environments. We detail differences in deployment models, considerations with IPv6 environments, and tradeoffs. If you are interested in deployment models for IPv4-only networks, they are discussed in the post <a href="https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/">Deployment models for AWS Network Firewall</a>.</p> 
<p><a href="https://aws.amazon.com/about-aws/whats-new/2023/01/aws-network-firewall-ipv6-support/">IPv6 dual stack support</a> was added to AWS Network Firewall in January 2023, and <a href="https://aws.amazon.com/about-aws/whats-new/2023/04/aws-network-firewall-ipv6-only-subnets/">IPv6-only support</a> was added in April 2023. You can find more resources on IPv6 best practices, reference architectures, and documentation in the re:Post article <a href="https://repost.aws/articles/AREREdDTYdSh2-NMBqQJS4wQ/get-started-with-ipv6-on-aws-resources-content">Get started with IPv6 on AWS</a>.</p> 
<h2>IPv6 deployment models on AWS</h2> 
<p>When deploying IPv6 environments on <a href="https://aws.amazon.com/">Amazon Web Services (AWS)</a>, there are two main options for deployment: dual-stack and IPv6-only. In a dual-stack model, your environment supports both IPv4 and IPv6, while IPv6-only means that workloads have only IPv6 addresses configured.</p> 
<h1>Dual-stack deployment model</h1> 
<p>A dual-stack deployment model has an environment that supports both IPv4 and IPv6 traffic. Resources in your environment have both IPv4 and IPv6 addresses. A benefit of deploying dual-stack environments is the native backward compatibility with resources not supporting IPv6. For example, you may have clients that are IPv4-only, so native IPv4 connectivity to a dual-stack endpoint is possible on IPv4.</p> 
<h1>IPv6-only deployment model</h1> 
<p>In an IPv6-only environment, your resources have only IPv6 addresses and can only send and receive IPv6 traffic. An IPv6-only model can be selected after ensuring clients, AWS services, and third-party resources support IPv6 or a backward-compatibility layer is configured. For example, you can deploy IPv6-only <a href="https://aws.amazon.com/ec2/">Amazon Elastic Compute Cloud (Amazon EC2)</a> instances for a service in your virtual private cloud (VPC) and use a dual-stack load balancer to allow both IPv4 and IPv6 clients to access the service.</p> 
<p>Let’s review the common architecture patterns for AWS Network Firewall in IPv6-enabled environments.</p> 
<h2>Internet ingress inspection architecture patterns</h2> 
<p>In this section will explore how you can use AWS Network Firewall to inspect IPv6 internet ingress traffic. AWS Network Firewall can be used in a distributed or centralized model.</p> 
<h2>IPv6 distributed -internet ingress inspection</h2> 
<p>In the first scenario, we focus on external clients connecting to an application hosted on AWS and using IPv6, as shown in the following diagram (Figure 1).</p> 
<div class="wp-caption aligncenter" id="attachment_22124" style="width: 1265px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/05/22/Fig1-2.png" rel="noopener" target="_blank"><img alt="Figure 1: IPv6 internet ingress pattern" class="wp-image-22124 size-full" height="819" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/05/22/Fig1-2.png" width="1255" /></a>
 <p class="wp-caption-text" id="caption-attachment-22124">Figure 1: IPv6 internet ingress pattern</p>
</div> 
<p>(1) The dual stack Application Load Balancer maintains both A (IPv4) and AAAA (IPv6) fully qualified domain name (FQDN) records in <a href="https://aws.amazon.com/route53/">Amazon Route 53</a> to allow both IPv4 and IPv6 clients access to your application. You can create a Route 53 public hosted zone for your custom DNS names, create both A and AAAA alias records, and point them to the Application Load Balancer FQDN.</p> 
<p>(2) Your IPv4 and IPv6 clients receive the A or AAAA records, depending on the DNS queries they perform, based on their capabilities.</p> 
<p>(3) Internet ingress traffic is inspected by AWS Network Firewall with endpoints in dual-stack subnets.</p> 
<p>(4) Ingress routing functionality uses <a href="https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html#gateway-route-tables">gateway route tables</a>, allowing AWS Network Firewall to be inserted in the traffic path between the internet gateway and your application.</p> 
<p>(5) The traffic destination is your Elastic Load Balancer (ELB), and the firewall acts as a bump-in-the-wire model inspection resource, transparent to the client and application.</p> 
<p>(6) After inspection, traffic is forwarded to your load balancer without any changes in IP addresses (IPv4 or IPv6).</p> 
<p>(7) The ELB then sends traffic to your backend EC2 instances on the protocol version they support, either IPv4 or IPv6. A dual-stack load balancer supports both IPv4 target groups and IPv6 target groups.</p> 
<p>This model supports both Application and Network Load Balancers, and AWS Network Firewall can filter both IPv4 and IPv6 traffic.</p> 
<h3>AWS Network Firewall configuration</h3> 
<p>The following example shows standard stateful rules allowing HTTP port 443 to the ELB subnet and blocking HTTP port 80 traffic. Customers can also use <a href="https://docs.aws.amazon.com/network-firewall/latest/developerguide/suricata-examples.html">Suricata compatible rules</a>.</p> 
<div class="wp-caption aligncenter" id="attachment_21945" style="width: 1513px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/05/20/Fig2-1.png" rel="noopener" target="_blank"><img alt="Figure 2: Standard stateful rules configuration allowing HTTP port 443" class="wp-image-21945 size-full" height="213" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/05/20/Fig2-1.png" width="1503" /></a>
 <p class="wp-caption-text" id="caption-attachment-21945">Figure 2: Standard stateful rules configuration allowing HTTP port 443</p>
</div> 
<p>An example of a Suricata rule to allow ingress traffic to IPv6 Classless Inter-Domain Routing (CIDR) is shown in the following code snippet:</p> 
<p><code>pass tcp any any -&gt; [2600:1f14:f25:df00::/56] 443 (flow:established,to_server; msg:"Allow TCP 443 traffic"; sid: 1000005; rev:1;)</code></p> 
<h2>IPv6 centralized internet ingress inspection</h2> 
<p>Centralized internet ingress patterns are limited in IPv6 to dual-stack applications and Network Load Balancers with IPv4 targets in other VPCs. Currently, you cannot configure IP-type target groups containing IPv6 addresses of your workloads residing in other VPCs or in on-premises networks, even if private connectivity is configured using VPC peering, <a href="https://aws.amazon.com/transit-gateway/">AWS Transit Gateway</a>, <a href="https://aws.amazon.com/cloud-wan/">AWS Cloud WAN</a>, or <a href="https://aws.amazon.com/directconnect/">AWS Direct Connect</a>.</p> 
<p>However, if you wish to have IPv6 targets and IPv6 services, you can centralize internet ingress using dual-stack Elastic Load Balancers and IPv6 targets by using IPv6 IP-type targets as the dual-stack <a href="https://aws.amazon.com/privatelink/">AWS PrivateLink</a> endpoints in the centralized ingress VPC. The PrivateLink endpoints allow your front-end Application or Network Load Balancer to connect using IPv6 to services in other VPCs or your on-premises network.</p> 
<div class="wp-caption aligncenter" id="attachment_21956" style="width: 1246px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/05/20/Fig3.png" rel="noopener" target="_blank"><img alt="Figure 3: IPv6 centralized internet ingress with PrivateLink pattern" class="wp-image-21956 size-full" height="819" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/05/20/Fig3.png" width="1236" /></a>
 <p class="wp-caption-text" id="caption-attachment-21956">Figure 3: IPv6 centralized internet ingress with PrivateLink pattern</p>
</div> 
<h3>AWS Network Firewall configuration</h3> 
<p>When you configure Network Firewall policy variables for a firewall policy, you can add one or more IPv4 or IPv6 addresses in CIDR notation to override the default value of Suricata HOME_NET. If your firewall is deployed using a centralized deployment model, you might want to override HOME_NET with the CIDRs of your inspection VPC, as shown in the following diagram (Figure 4).</p> 
<div class="wp-caption aligncenter" id="attachment_21959" style="width: 1117px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/05/20/Fig4.png" rel="noopener" target="_blank"><img alt="Figure 4: AWS Network Firewall HOME_NET configuration" class="wp-image-21959 size-full" height="690" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/05/20/Fig4.png" width="1107" /></a>
 <p class="wp-caption-text" id="caption-attachment-21959">Figure 4: AWS Network Firewall HOME_NET configuration</p>
</div> 
<h2>Internet egress inspection architecture patterns</h2> 
<p>As you deploy workloads in AWS, many resources will need access to the internet. This traffic often must be inspected. This section will explore two options for inspecting egress to the internet for your AWS resources.</p> 
<h2>IPv6 distributed internet egress inspection</h2> 
<p>Workloads deployed in dual-stack or IPv6-only subnets connect to internet endpoints using either IPv4 or IPv6, and the traffic can be inspected using AWS Network Firewall for both IPv4 and IPv6. As shown in the following diagram (Figure 5), we deploy AWS Network Firewall endpoints in a dual-stack subnet to allow traffic inspection of both IPv4 and IPv6 protocols.</p> 
<p>Filtering IPv6 egress traffic using AWS Network Firewall is possible today only for public subnets. Private IPv6 subnets are configured with a default route to the egress-only internet gateway, which doesn’t support ingress routing capabilities, to allow for symmetrical inspection of the return traffic through the firewall endpoints.</p> 
<div class="wp-caption aligncenter" id="attachment_22248" style="width: 1147px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/06/04/Fig5-2.png" rel="noopener" target="_blank"><img alt="Figure 5: IPv6 distributed internet egress inspection" class="wp-image-22248 size-full" height="804" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/06/04/Fig5-2.png" width="1137" /></a>
 <p class="wp-caption-text" id="caption-attachment-22248">Figure 5: IPv6 distributed internet egress inspection</p>
</div> 
<p>Using an internet gateway for egress means that in IPv6, the subnets are public. Therefore, appropriate restrictions should be applied on AWS Network Firewall for blocking internet originated ingress traffic to the workload subnets.</p> 
<h3>NAT64 and DNS64 to communicate with IPv4-only services</h3> 
<p>If you have subnets with IPv6-only workloads that needs to communicate with IPv4-only services on the Internet you can use AWS managed DNS64 together with NAT64. If there is no IPv6 address associated with the destination in the DNS record, the Route 53 Resolver synthesizes one by prepending the well-known prefix 64:ff9b::/96 (defined in <a href="https://datatracker.ietf.org/doc/html/rfc6052">RFC6052</a>), to the IPv4 address in the record. Your IPv6-only service sends network packets to the synthesized IPv6 address. You will then need to route this traffic through the NAT gateway, a Network Address Translation (NAT) service, which performs the necessary NAT64 translation on the traffic to allow IPv6 services in your subnet to access IPv4 services outside that subnet.</p> 
<p>You can enable or disable DNS64 on a subnet using the <code>aws ec2 modify-subnet-attribute</code> <a href="https://docs.aws.amazon.com/cli/latest/reference/ec2/modify-subnet-attribute.html">command</a> using the AWS CLI or with the VPC console by editing the subnet settings. NAT64 is automatically available on your existing NAT gateways or on any new NAT gateways you create. For more information see the DNS64 and NAT64 documentation.</p> 
<p>As shown in figure 5, firewall endpoint subnets have route for 64:ff9b::/96 pointing towards the NAT gateway, this is required for NAT64 to work.</p> 
<h3>AWS Network Firewall configuration</h3> 
<p>Example Suricata rule to block ingress traffic to IPv6 CIDR:</p> 
<p><code>drop ip any any -&gt; [2600:1f14:f25:df00::/56] any (flow:established,to_server; msg:"Deny IPv6 traffic"; sid: 1000006; rev:1;)</code></p> 
<p>Note that you can combine two models: IPv4 centralized egress and IPv6 distributed egress.</p> 
<h2>IPv6 centralized internet egress inspection</h2> 
<p>To implement the centralized IPv6 internet egress pattern, you must use NAT66 or IPv6-to-IPv6 Network Prefix Translation (NPTv6) capable firewalls or proxies, together with Gateway Load Balancer, deployed in two-arm mode. The Gateway Load Balancer two-arm deployment model is explained in the blog post <a href="https://aws.amazon.com/blogs/networking-and-content-delivery/best-practices-for-deploying-gateway-load-balancer/">Best practices for deploying Gateway Load Balancer</a>. When possible, we recommend that you implement a distributed egress pattern, discussed in a preceding section, to improve scalability and optimize data processing costs.</p> 
<h2>VPC-to-VPC traffic inspection deployment patterns</h2> 
<p>Let’s explore how you can use AWS Network Firewall to inspect IPv6 traffic between different environments or security zones on AWS. AWS Network Firewall can be used in a centralized or distributed model in this pattern. You can inspect IPv6 traffic within a VPC, for example, from a private subnet to a public subnet, or between VPCs.</p> 
<h2>Centralized VPC-to-VPC inspection pattern with Transit Gateway and AWS Cloud WAN</h2> 
<p>You can use a centralized VPC to deploy AWS Network Firewall and route traffic to the central firewall deployment using AWS Transit Gateway or AWS Cloud WAN. This model is discussed in the Centralized Deployment section of <a href="https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/">Deployment models for AWS Network Firewall</a> for Transit Gateway and <a href="https://aws.amazon.com/blogs/networking-and-content-delivery/inspecting-network-traffic-between-amazon-vpcs-with-aws-cloud-wan/">Inspecting network traffic between Amazon VPCs with AWS Cloud WAN</a>. To use this architecture in a dual-stack environment, you need Transit Gateway or AWS Cloud WAN dual-stack VPC attachment subnets, and you must update your route tables to include the appropriate IPv6 routes. The following diagram (Figure 6) shows an example of AWS Transit Gateway deployment:</p> 
<div class="wp-caption aligncenter" id="attachment_21965" style="width: 1261px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/05/20/Fig6.png" rel="noopener" target="_blank"><img alt="Figure 6: Centralized VPC-to-VPC inspection with Transit Gateway" class="wp-image-21965 size-full" height="754" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/05/20/Fig6.png" width="1251" /></a>
 <p class="wp-caption-text" id="caption-attachment-21965">Figure 6: Centralized VPC-to-VPC inspection with Transit Gateway</p>
</div> 
<p>The following diagram (Figure 7) shows an example of AWS Cloud WAN deployment in two AWS Regions. With stateful inspection, to maintain symmetry across Regions, we configured routing such that traffic traverses the firewalls in both source and destination Regions.</p> 
<div class="wp-caption aligncenter" id="attachment_21966" style="width: 1184px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/05/20/Fig7.png" rel="noopener" target="_blank"><img alt="Figure 7: Centralized VPC-to-VPC inspection with AWS Cloud WAN" class="wp-image-21966 size-full" height="829" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/05/20/Fig7.png" width="1174" /></a>
 <p class="wp-caption-text" id="caption-attachment-21966">Figure 7: Centralized VPC-to-VPC inspection with AWS Cloud WAN</p>
</div> 
<h2>Considerations</h2> 
<ul> 
 <li>There is no additional cost to enable dual-stack AWS Network Firewall endpoints.</li> 
 <li>There is no additional cost to use IPv6 in Amazon VPC.</li> 
 <li>Check out the reference architectures for more information on Dual stack and IPv6-only design patterns, view <a href="https://d1.awsstatic.com/architecture-diagrams/ArchitectureDiagrams/IPv6-reference-architectures-for-AWS-and-hybrid-networks-ra.pdf?did=wp_card&amp;trk=wp_card">Dual Stack and IPv6-only Amazon VPC Reference Architectures</a>.</li> 
 <li>For more information on IPv6 VPCs and subnets, view <a href="https://aws.amazon.com/about-aws/whats-new/2023/11/vpcs-subnets-support-more-sizes-ipv6-cidrs/">VPCs and subnets now support more sizes for IPv6 CIDRs</a>.</li> 
 <li>For general AWS Network Firewall best practices, view <a href="https://aws.github.io/aws-security-services-best-practices/guides/network-firewall/">AWS Network Firewall Best Practices Guide</a>.</li> 
 <li>For more information about IPv6 on AWS, view <a href="https://repost.aws/articles/AREREdDTYdSh2-NMBqQJS4wQ/get-started-with-ipv6-on-aws-resources-content">Get started with IPv6 on AWS – Resources &amp; Content</a>.</li> 
</ul> 
<h2>Conclusion</h2> 
<p>IPv6 adoption is increasing every year, and many AWS customers have reasons and requirements for adopting IPv6 in their environments. Ensuring network security through traffic inspection remains a top priority for IPv4-only, dual-stack, and IPv6-only environments. AWS Network Firewall provides the flexibility and capability to inspect both protocol stacks at no additional cost. To learn about additional IPv6 security and monitoring considerations, read the IPv6 section of the whitepaper <a href="https://docs.aws.amazon.com/whitepapers/latest/ipv6-on-aws/ipv6-security-and-monitoring-considerations.html">Best practices for adopting and designing IPv6-based networks on AWS</a>.</p> 
<p><em>An update was made on June 4, 2024: An earlier version of this post omitted the possibility to use DNS64 and NAT64. The post has been updated to describe the usage of DNS64 and NAT64 for internet egress inspection to IPv4-only targets.</em></p> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="nick-kniveton-square.jpg"><img alt="Nick Kniveton" class="alignleft wp-image-1288 size-thumbnail" height="125" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/05/20/nick-kniveton-square.jpg" width="125" /></p> 
 <h3 class="lb-h4">Nick Kniveton</h3> 
 <p>Nick Kniveton is a solutions architect at Amazon Web Services (AWS) specializing in Networking and Resilience. Nick works with public sector customers to design and operate highly resilient and scalable networking architectures for AWS and hybrid environments.</p> 
</div> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="Tushar-Jagdale-1.jpg"><img alt="Tushar Jagdale" class="alignleft wp-image-1288 size-thumbnail" height="125" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/15/Tushar-Jagdale-1.jpg" width="125" /></p> 
 <h3 class="lb-h4">Tushar Jagdale</h3> 
 <p>Tushar is a Specialist Solutions Architect focused on Networking at AWS, where he helps customers build and design scalable, highly-available, secure, resilient and cost effective networks. He has over 15 years of experience building and securing Data Center and Cloud networks.</p> 
</div>
