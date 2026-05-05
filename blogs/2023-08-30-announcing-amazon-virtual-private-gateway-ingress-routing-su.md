---
title: "Announcing Amazon Virtual Private Gateway Ingress Routing support for Gateway Load Balancer"
url: "https://aws.amazon.com/blogs/networking-and-content-delivery/announcing-amazon-virtual-private-gateway-ingress-routing-support-for-gateway-load-balancer/"
date: "Wed, 30 Aug 2023 19:02:33 +0000"
author: "Tushar Jagdale"
feed_url: "https://aws.amazon.com/blogs/networking-and-content-delivery/tag/aws-network-firewall/feed/"
---
<p>Today, on 30th August 2023, AWS launched a new enhancement to the <a href="https://aws.amazon.com/vpc/">Amazon Virtual Private Cloud (Amazon VPC)</a> <a href="https://aws.amazon.com/blogs/aws/new-vpc-ingress-routing-simplifying-integration-of-third-party-appliances/">Ingress Routing</a> feature. With this enhancement, customers can now specify a <a href="https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/getting-started.html">Gateway Load Balancer Endpoint (GWLBE)</a> as the next-hop in the virtual private gateway (VGW) route table. This allows customers to inspect their traffic coming into AWS via <a href="https://docs.aws.amazon.com/vpn/latest/s2svpn/VPC_VPN.html">AWS Site-to-Site VPN</a> or <a href="https://aws.amazon.com/directconnect/">AWS Direct Connect</a> using <a href="https://aws.amazon.com/network-firewall/">AWS Network Firewall</a> (which is powered by <a href="https://aws.amazon.com/elasticloadbalancing/gateway-load-balancer/">Gateway Load Balancer</a> (GWLB)) or third-party firewalls running in AWS. This enhancement simplifies deployment and helps ensure consistent traffic inspection at entry points into customers’ Amazon VPCs.</p> 
<p>In this post, we explain how the newly launched feature works, how to configure it, describe common deployment scenarios, and discuss considerations while using this feature.</p> 
<h2><strong>What’s new</strong></h2> 
<p>There are now five ways customers can connect their environments to a VPC via VGW, as shown in Figure 1.</p> 
<ul> 
 <li>Figure 1a shows an on-premises corporate data center connecting to VPC using Direct Connect (DX) Private Virtual Interface (VIF) to VGW.</li> 
 <li>Figure 1b shows an on-premises corporate data center connecting to VPCs using Direct Connect (DX) Private Virtual Interface (VIF) via Direct Connect Gateway (DXGW) to VGW.</li> 
 <li>Figure 1c shows an on-premises corporate data center connecting to VPC using IPSec Site-to-site VPN over Direct Connect (DX) Public Virtual Interface (VIF) to VGW.</li> 
 <li>Figure 1d shows an on-premises corporate data center connecting to VPC using IPSec Site-to-site VPN over the Internet.</li> 
 <li>Figure 1e shows an IPSec Site-to-site VPN connectivity between two AWS environments.</li> 
</ul> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/08/18/Fig1-2.png" rel="noopener" target="_blank"><img alt="Figure 1: VGW connectivity options" class="aligncenter wp-image-17884 size-full" height="654" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/08/18/Fig1-2.png" width="1576" /></a></p> 
<p style="text-align: center;"><em>Figure 1: VGW connectivity options</em></p> 
<p>Prior to this feature, it was not possible to send the incoming traffic arriving at VGW directly to a GWLB endpoint or GWLB for inspection. As a workaround, customers used an <a href="https://aws.amazon.com/transit-gateway/">AWS Transit Gateway</a> and an inspection VPC. This incurred additional costs, management overhead, and latency. Now, with the VGW route table supporting the ingress routing feature for GWLBE, customers can set GWLBE as a next-hop in the VGW route table. This allows customers to steer traffic directly to GWLBE and GWLB, thereby reducing costs, management overhead, and latency. This new feature works across all scenarios in Figure 1.</p> 
<h2><strong>Feature deep dive</strong></h2> 
<p>Let’s describe how this feature works and how to configure it in a step-by-step manner. We will do this for the three most common deployment scenarios:</p> 
<ul> 
 <li>Scenario 1: Using VGW Ingress routing with GWLB and third-party virtual appliances.</li> 
 <li>Scenario 2: Using VGW Ingress routing with AWS Network Firewall.</li> 
 <li>Scenario 3: Using Ingress routing with an <a href="https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html">Internet Gateway (IGW)</a> and VGW.</li> 
</ul> 
<h2><strong>Scenario 1: Using VGW Ingress routing with GWLB and third-party virtual appliances</strong></h2> 
<p>In this scenario (Figure 2), on-premises connectivity to the Amazon VPC is via VPN or Direct Connect (DX). The intent is to inspect all the traffic using third-party virtual appliances in the Security VPC before it reaches the protected application subnet (10.2.2.0/24).</p> 
<p><img alt="" class="alignnone wp-image-18321 size-full" height="760" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/09/08/figure-2-updated.png" width="1013" /></p> 
<p style="text-align: center;"><em>Figure 2: VGW Ingress routing with GWLB</em></p> 
<h3><strong>How does the feature work?</strong></h3> 
<p>The following steps describe a packet walkthrough when the traffic originates from the remote site either via a VPN or DX connection toward the application hosted in the protected VPC:</p> 
<p>(1) The remote network device conducts a route lookup for the destination in the VPC and selects the VPN or DX.</p> 
<p>(2) When the packet arrives at the VGW a route lookup occurs in the VPC Ingress route table that is associated with the VGW, a more specific route for the application subnet is pointing toward the Gateway Load Balancer Endpoint (10.2.2.0/24 -&gt; vpce-id).</p> 
<p>(3) Traffic, routed through the GWLBE, is delivered securely and privately to the GWLB using <a href="https://docs.aws.amazon.com/vpc/latest/privatelink/vpce-gateway-load-balancer.html">AWS PrivateLink</a>.</p> 
<p>(4) GWLB sends the traffic to the third-party virtual appliances, such as next generation firewalls (NGFW), or intrusion detection and prevention systems (IDS/IPS). The third-party appliances inspect the traffic, and then send it back to the GWLBE.</p> 
<p>(5) GWLBE sends the traffic to the protected application subnet (10.2.2.0/24).</p> 
<p>Reverse traffic (not shown) follows the same path in the reverse direction.</p> 
<h3><strong>How to configure this feature</strong></h3> 
<p>Before configuring this feature, make sure you complete the following prerequisite actions:</p> 
<ul> 
 <li>Set-up your VPN/DX, Security VPCs, and Protected VPC.</li> 
 <li>Configure your Security VPC (i.e. the one containing GWLB and targets) with: 
  <ul> 
   <li>A <a href="https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/create-load-balancer.html">GWLB</a> with a target group that has third-party virtual appliances.</li> 
   <li>An endpoint service created for the GWLB: 
    <ul> 
     <li>Create <a href="https://docs.aws.amazon.com/vpc/latest/privatelink/create-gateway-load-balancer-endpoint-service.html">VPC Endpoint Service</a> and associate it with the GWLB. Note the endpoint service name which starts with “com.amazonaws.vpce.”.</li> 
     <li>If the Protected VPC is in a different AWS account, then it should be listed in the allowed principal list for the endpoint service (see <a href="https://docs.aws.amazon.com/vpc/latest/privatelink/configure-endpoint-service.html#add-remove-permissions">manage permissions</a>).</li> 
    </ul> </li> 
  </ul> </li> 
 <li>Configure your Protected VPC (i.e. the one containing GWLBEs) with: 
  <ul> 
   <li><a href="https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-load-balancer-endpoints.html">A VPC Endpoint</a> (GWLBE) pointing to the same service name as identified on the endpoint service above. Note that when creating the endpoint, only the Availability Zones (AZs) that are in common are available for selection.</li> 
  </ul> </li> 
</ul> 
<p>Now that the prerequisites are ready, let’s take a look at how to configure this feature using either <a href="https://aws.amazon.com/cli/">AWS Command Line Interface (AWS CLI)</a> or the <a href="https://aws.amazon.com/console/">AWS Management Console</a>. You can use either method, as per your preference.</p> 
<h3><strong>Configure the feature using AWS CLI</strong></h3> 
<p><strong>Forward Traffic</strong>: First, let’s set up routing tables for forward traffic (traffic going from corporate data center to the protected subnet).</p> 
<ul> 
 <li>Create a new route table to be the ingress route table, and associate it with the VGW:</li> 
</ul> 
<pre><code class="lang-bash">aws ec2 create-route-table \
     --vpc-id vpc-038d18783f44f9148
{
    "RouteTable": {
        "Associations": [],
        "PropagatingVgws": [],
        "RouteTableId": "rtb-0cc22db998e104134",
        "Routes": [
            {
                "DestinationCidrBlock": "10.2.0.0/16",
                "GatewayId": "local",
                "Origin": "CreateRouteTable",
                "State": "active"
            }
        ],
        "Tags": [],
        "VpcId": "vpc-038d18783f44f9148",
        "OwnerId": "795662187403"
    }
}


$ aws ec2 associate-route-table \
--route-table-id rtb-0cc22db998e104134 \
--gateway-id vgw-0763bd41338524306
{
    "AssociationId": "rtbassoc-092b8c045de3f5812",
    "AssociationState": {
        "State": "associated"
    }
}</code></pre> 
<ul> 
 <li>For each application subnet you want to protect, create a route in the VGW Ingress route table, with the subnet CIDR as the route destination, and the GWLBE as the route target. This steers all incoming traffic destined to that subnet to the VPCe (GWLBE).</li> 
</ul> 
<pre><code class="lang-bash">$ aws ec2 create-route \
--route-table-id rtb-0cc22db998e104134 \
--destination-cidr-block 10.2.2.0/24 \
--vpc-endpoint-id vpce-0c3ef88210b4cea51
{
    "Return": true
}</code></pre> 
<p><strong>Reverse Traffic</strong>: Now, let’s set-up routing tables for reverse traffic, the traffic going from the Protected Subnet to your corporate data center.</p> 
<ul> 
 <li>First, get the route table associated with your application’s private subnet.</li> 
</ul> 
<pre><code class="lang-bash">$ aws ec2 describe-route-tables \
        --filters Name=association.subnet-id,Values=subnet-0e23061abca5265eb
{
    "RouteTables": [
        {
            "Associations": [
                {
                    "Main": false,
                    "RouteTableAssociationId": "rtbassoc-0fe664c250d902312",
                    "RouteTableId": "rtb-0767064621a86a46e",
                    "SubnetId": "subnet-0e23061abca5265eb",
                    "AssociationState": {
                        "State": "associated"
                    }
                }
            ],

# output truncated</code></pre> 
<ul> 
 <li>For each application’s private subnet, create the route for on-premises CIDR and point it toward the VPCe (GWLBE) in the corresponding route table.</li> 
</ul> 
<pre><code class="lang-bash">$ aws ec2 create-route \
--route-table-id rtb-0767064621a86a46e \
--destination-cidr-block 172.16.0.0/16 \
--vpc-endpoint-id vpce-0c3ef88210b4cea51

{
    "Return": true
}</code></pre> 
<ul> 
 <li>In the route table associated with the GWLBE subnet, point the route for remote CIDR toward the VGW:</li> 
</ul> 
<pre><code class="lang-bash">$ aws ec2 create-route \
--route-table-id rtb-095fd6a71689a0b4f \
--destination-cidr-block 172.16.0.0/16 \
--gateway-id vgw-0763bd41338524306
{
    "Return": true
}</code></pre> 
<h3><strong>Configure the feature using the Console</strong></h3> 
<p><strong>Forward Traffic</strong>: First, let’s set-up routing tables for forward traffic, the traffic going from your corporate data center to the Protected Subnet.</p> 
<p>Create a new route table, as shown in Figure 3a.</p> 
<p><img alt="" class="aligncenter size-full wp-image-17870" height="667" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/08/18/Fig3.png" width="828" /></p> 
<p style="text-align: center;"><em>Figure 3a: Create route table</em></p> 
<p>Associate the newly created route table with the VGW, as shown in Figure 3b. This is the ingress route table.</p> 
<p><img alt="" class="aligncenter size-full wp-image-17871" height="532" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/08/18/Fig3b.png" width="1589" /></p> 
<p style="text-align: center;"><em>Figure 3b: Associate route table with VGW</em></p> 
<p>Select the gateway to be associated with the route table, as shown in Figure 3c.</p> 
<p><img alt="" class="aligncenter size-full wp-image-17872" height="602" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/08/18/Fig3c.png" width="1002" /></p> 
<p style="text-align: center;"><em>Figure 3c: Select gateway to associate with the route table</em></p> 
<p>Verify that the gateway has been associated with the route table, as shown in Figure 3d.</p> 
<p><img alt="" class="aligncenter size-full wp-image-17873" height="585" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/08/18/Fig3d.png" width="1641" /></p> 
<p style="text-align: center;"><em>Figure 3d: Verify gateway association</em></p> 
<p>In the VGW ingress route table, point the route for the application subnet toward the GWLBE (note that in the route table the next-hop is shown as vpce-id), as shown in Figure 3e.</p> 
<p><img alt="" class="aligncenter size-full wp-image-17874" height="489" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/08/18/Fig3e.png" width="1377" /></p> 
<p style="text-align: center;"><em>Figure 3e: Configure route for the protected subnet</em></p> 
<p>Add the next-hop for the protected subnet as GWLBE in the route table, as shown in Figure 3f.</p> 
<p><img alt="" class="aligncenter size-full wp-image-17875" height="318" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/08/18/Fig3f.png" width="1378" /></p> 
<p style="text-align: center;"><em>Figure 3f: Target in the VGW Ingress route table</em></p> 
<p>Verify that the route has been added successfully, as shown in Figure 3g.</p> 
<p><img alt="" class="aligncenter size-full wp-image-17877" height="677" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/08/18/Fig3g.png" width="1584" /></p> 
<p style="text-align: center;"><em>Figure 3g: Route entry in the VGW Ingress route table</em></p> 
<p><strong>Reverse Traffic</strong>: Now, let’s set-up route tables for reverse traffic, the traffic going from the Protected Subnet to Corporate Data Center.</p> 
<p>Make sure the application subnet has the reverse route pointing toward the GWLB Endpoint as shown in Figure 3h.</p> 
<p><img alt="" class="aligncenter size-full wp-image-17878" height="641" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/08/18/Fig3h.png" width="1595" /></p> 
<p style="text-align: center;"><em>Figure 3h: Application subnet route table</em></p> 
<p>The subnet hosting the VPCe (GWLBE) should have a route for the remote site pointing toward the VGW, as shown in Figure 3i.</p> 
<p><img alt="" class="aligncenter size-full wp-image-17879" height="668" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/08/18/Fig3i.png" width="1591" /></p> 
<p style="text-align: center;"><em>Figure 3i: Firewall subnet route table</em></p> 
<h2>Scenario 2: Using VGW Ingress routing with AWS Network Firewall</h2> 
<p>In this scenario (Figure 4), on-premises connectivity to the Amazon VPC is via VPN. The intent is to inspect all traffic using AWS Network Firewall before it reaches the protected application subnet (10.2.2.0/24). In this scenario, we are inserting Network Firewall (which is powered by GWLB) in the traffic flow. When a new network firewall is created, a GWLB Endpoint gets created in that subnet, in the routing table you will see vpce-id as the target. For details on how to configure Network Firewall, refer to this <a href="https://docs.aws.amazon.com/network-firewall/latest/developerguide/getting-started.html">documentation</a>. Note that customers do not need to manage the underlying GWLB used by our AWS Network Firewall service.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/08/18/Fig4-1.png" rel="noopener" target="_blank"><img alt="Figure 4: VGW Ingress Routing with Network Firewall architecture" class="aligncenter wp-image-17880 size-full" height="742" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/08/18/Fig4-1.png" width="928" /></a></p> 
<p style="text-align: center;"><em>Figure 4: VGW Ingress Routing with Network Firewall architecture</em></p> 
<h3><strong>How does the feature work?</strong></h3> 
<p>The following steps describe a packet walkthrough when the traffic originates from the remote site via a VPN toward an application hosted in the protected VPC:</p> 
<p>(1) The remote network device conducts a route lookup for the destination in VPC and selects the VPN.</p> 
<p>(2) When the packet arrives at the VGW a route lookup occurs in the VPC Ingress route table that is associated with the VGW, a more specific route for the application subnet is pointing toward the Gateway Load Balancer Endpoint (10.2.2.0/24 -&gt; vpce-id).</p> 
<p>(3) Traffic, routed through a firewall endpoint, is delivered securely and privately to Network Firewall using <a href="https://docs.aws.amazon.com/vpc/latest/privatelink/vpce-gateway-load-balancer.html">AWS PrivateLink</a>.</p> 
<p>(4) Network Firewall performs the traffic inspection, and it sends the traffic back to the firewall endpoint.</p> 
<p>(5) Firewall Endpoint sends the traffic to the protected application subnet (10.2.2.0/24).</p> 
<p>Reverse traffic (not shown) follows the same path in the reverse direction.</p> 
<h2><strong>Scenario 3: Using Ingress routing with IGW and VGW</strong></h2> 
<p>There may be a scenario where you want to use ingress routing capability to protect an application subnet for both traffic coming via VGW and IGW. The VPC Ingress Routing feature, which prioritizes the longest prefix match, makes this possible. Figure 5 shows an example scenario where on-premises Corporate Data Center is connected to the AWS VPC via VPN or DX, and the VPC is also connected to the Internet using IGW.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/09/08/updated-image-to-announcing-1.png" rel="noopener" target="_blank"><img alt="" class="aligncenter wp-image-18319 size-full" height="754" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/09/08/updated-image-to-announcing-1.png" width="1214" /></a></p> 
<p style="text-align: center;"><em>Figure 5: Ingress routing IGW, VGW, and GWLB</em></p> 
<h3><strong>How does the feature work with VGW?</strong></h3> 
<p>The following steps describe a packet walkthrough when the traffic originates from the remote site either via a VPN or DX toward the application hosted in the protected VPC:</p> 
<p>(1) The remote network device conducts a route lookup for the destination in a VPC and selects the VPN or DX.</p> 
<p>(2) When the packet arrives at the VGW a route lookup occurs in the VPC Ingress route table that is associated with the VGW, a more specific route for the application subnet is pointing toward the Gateway Load Balancer Endpoint (10.2.2.0/24 -&gt; vpce-id).</p> 
<p>(3) Traffic, routed through GWLBE, is delivered securely and privately to GWLB using <a href="https://docs.aws.amazon.com/vpc/latest/privatelink/vpce-gateway-load-balancer.html">AWS PrivateLink</a>.</p> 
<p>(4) The third-party appliances inspect the traffic, and then send it back to the GWLBE.</p> 
<p>(5) GWLBE sends the traffic to the protected application subnet (10.2.2.0/24).</p> 
<h3><strong>How does the feature work with IGW?</strong></h3> 
<p>The following steps describe a packet walkthrough when the traffic originates from a client on the Internet and is sent to an application hosted in a VPC via an IGW. Note that this capability (IGW Ingress Routing support for GWLB) has been available since 2020.</p> 
<p>A. The remote client conducts a route lookup for the destination in a VPC and reaches IGW.</p> 
<p>B. When the packet arrives at the IGW a route lookup occurs in the VPC Ingress route table that is associated with the IGW, a more specific route for the application subnet is pointing toward the Gateway Load Balancer Endpoint (10.2.2.0/24 -&gt; vpce-id).</p> 
<p>C. Traffic, routed through GWLBE, is delivered securely and privately to GWLB using <a href="https://docs.aws.amazon.com/vpc/latest/privatelink/vpce-gateway-load-balancer.html">AWS PrivateLink</a>.</p> 
<p>D. The third-party appliances inspect the traffic, and then send it back to the GWLBE.</p> 
<p>E. GWLBE sends the traffic to the protected application subnet (10.2.2.0/24).</p> 
<h2><strong>Considerations</strong></h2> 
<p>Consider the following requirements when preparing to deploy VGW Ingress Routing using GWLBE as the next-hop:</p> 
<ul> 
 <li>The GWLBE used as the next-hop target must be deployed in the same VPC that is associated with the VGW.</li> 
 <li>The destination subnets must belong to a VPC associated with the VGW.</li> 
 <li>You cannot enable VGW route propagation for the same route table you use as the VGW Ingress route table.</li> 
 <li>Using VGW Ingress routing functionality you can also add IPv6 dual-stack subnet as well as IPv6 only subnets as targets</li> 
 <li>Site-to-Site VPN connections on a VGW do not support IPv6. However, Site-to-Site VPN supports IPv6 traffic for VPN connections attached to Transit Gateway.</li> 
 <li>Connectivity to VPC via VGW is 1:1, which means one VGW connects to only one VPC. This feature applies only in these scenarios. If you have multiple VPCs and are looking for centralized inspection models using hub and spoke topology, then see this <a href="https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/">post</a>.</li> 
</ul> 
<h2><strong>Conclusion:</strong></h2> 
<p>In this post, we saw how to steer traffic coming from on-premises using VPN or DX via a VGW to a GWLBE or a Network Firewall Endpoint using the new VGW Ingress routing enhancement. This enhancement allows customers to insert third-party appliances or AWS Network Firewall seamlessly before the traffic reaches the application workloads.</p> 
<p>There are no additional costs to use Amazon VPC ingress routing. It is available in all <a href="https://aws.amazon.com/about-aws/global-infrastructure/regions_az/">AWS Regions</a> (including <a href="https://aws.amazon.com/govcloud-us/">AWS GovCloud</a>) and you can start using it today.</p> 
<p>You can learn more about gateway routing tables in the updated <a href="https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html#gateway-route-table">VPC documentation</a>. For more details on GWLB, refer to <a href="https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/introduction.html">GWLB documentation</a> and <a href="https://aws.amazon.com/blogs/networking-and-content-delivery/category/networking-content-delivery/elastic-load-balancing/gateway-load-balancer/">GWLB posts</a>.</p> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="Milind_K.jpg"><img alt="Milind Kulkarni" class="alignleft wp-image-1288 size-thumbnail" height="96" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2020/11/16/Milind_K.jpg" width="120" /></p> 
 <h3 class="lb-h4">Milind Kulkarni</h3> 
 <p>Milind is a Principal Product Manager at Amazon Web Services (AWS). He has over 20 years of experience in networking, data center architectures, SDN/NFV, and cloud computing. He is a co-inventor of nine US Patents and has co-authored three IETF Standards.</p> 
</div> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="Tushar-Jagdale-1.jpg"><img alt="Tushar Jagdale" class="alignleft wp-image-1288 size-thumbnail" height="125" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/15/Tushar-Jagdale-1.jpg" width="125" /></p> 
 <h3 class="lb-h4">Tushar Jagdale</h3> 
 <p>Tushar is a Specialist Solutions Architect focused on Networking at AWS, where he helps customers build and design scalable, highly-available, secure, resilient and cost effective networks. He has over 15 years of experience building and securing Data Center and Cloud networks.</p> 
</div>
