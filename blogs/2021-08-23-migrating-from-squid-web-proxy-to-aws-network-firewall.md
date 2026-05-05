---
title: "Migrating from Squid Web Proxy to AWS Network Firewall"
url: "https://aws.amazon.com/blogs/networking-and-content-delivery/migrating-from-squid-web-proxy-to-aws-network-firewall/"
date: "Mon, 23 Aug 2021 16:21:35 +0000"
author: "Shiva Vaidyanathan"
feed_url: "https://aws.amazon.com/blogs/networking-and-content-delivery/tag/aws-network-firewall/feed/"
---
<h2>Introduction</h2> 
<p>Regardless of size or industry, it’s common for organizations to have security and compliance rules for securing internet-bound traffic. AWS customers need control over, and the ability to filter, requests that are initiated by resources in private and public subnets and sent to the internet. This is also known as “egress filtering.”</p> 
<p>In AWS, you have many options for egress traffic filtering. <a href="https://aws.amazon.com/vpc/">Amazon Virtual Private Cloud</a> (VPC) network security features, such as <a href="https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html">security groups</a> and <a href="https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html">network ACLs</a> help build a layered network defense for your VPC. <a href="https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html">NAT gateways</a> are used to allow resources in private subnets egress access to the internet. However, these services don’t provide the advanced traffic filtering features that your business might require, such as trojan, web, and malware filtering. To meet these needs, you may decide to implement third-party <a href="https://aws.amazon.com/marketplace/search?searchTerms=firewall">Independent Software Vendor software</a> solutions on EC2 instances. You could use iptables rules to restrict outbound traffic based on a predefined destination port or IP address. You could also use an open source <a href="http://www.squid-cache.org/">Squid proxy</a> solution and implement it as an explicit proxy. (The blog post, <a href="https://aws.amazon.com/blogs/security/how-to-add-dns-filtering-to-your-nat-instance-with-squid/">How to add DNS filtering to your NAT instance with Squid</a>, walks through the implementation of a Squid proxy solution on AWS.) The Squid proxy can restrict both HTTP and HTTPS outbound traffic to a given set of internet domains names (for example, www.example.com). The previously mentioned methods require customers to manage the scaling, patching, and high availability of the underlying resources.</p> 
<p><a href="https://aws.amazon.com/network-firewall/">AWS Network Firewall</a> provides advanced egress filtering capability and, at the same time, can be simple to deploy and operate. The AWS Network Firewall is a managed service that scales automatically with your network traffic, and offers built-in redundancies designed to provide high availability. It provides fine-grained network protections for resources that exist in Virtual Private Clouds (VPCs). After updating your VPC routing configuration, the AWS Network Firewall ensures that your traffic is inspected. You can set up monitoring and logging as required. AWS Network Firewall also offers a flexible rules engine as <a href="https://docs.aws.amazon.com/network-firewall/latest/developerguide/stateless-rule-groups-5-tuple.html">stateless</a> and <a href="https://docs.aws.amazon.com/network-firewall/latest/developerguide/stateful-rule-groups-ips.html">stateful</a> rule groups that give you the ability to write tens of thousands of firewall rules for granular policy enforcement. It supports inbound and outbound web filtering for unencrypted web traffic. For encrypted web traffic, AWS Network Firewall inspects the domain name provided by the Server Name Indicator (SNI) during the Transport Layer Security (TLS) handshake. Also, it offers an <a href="https://aws.amazon.com/blogs/opensource/scaling-threat-prevention-on-aws-with-suricata/">Intrusion Prevention System</a> (IPS), which provides inspection to help you identify and block vulnerability exploits.</p> 
<p>In this post, we highlight common architecture patterns that customers use with a Squid proxy solution in their AWS environments. We then show how to migrate this setup to a solution using AWS Network Firewall. Finally, we provide a tool that can help this migration process and provide <a href="https://aws.amazon.com/cloudformation/">AWS CloudFormation</a> templates to spin up the infrastructure required.</p> 
<h2>Key terms and concepts</h2> 
<p>Let’s review some of the key terms and concepts that are used in this blog:</p> 
<p><strong>Network Firewall</strong> – An AWS resource that provides firewall and intrusion prevention services for your VPC, at scale, and with support for tens of thousands of rules.<br /> <strong>Network Firewall policy</strong> – An AWS resource that provides network traffic behavior for the firewall. It defines a reusable set of stateful and stateless rule groups along with some policy level behavior settings.<br /> <strong>Network Firewall rule group</strong> – An AWS resource that defines a set of rules to match against VPC traffic, and the actions to take when Network Firewall finds a match.<br /> <strong>Suricata</strong> – <a href="https://suricata.io/">Suricata</a> is an open source threat detection engine that provides capabilities including intrusion detection, intrusion prevention and network security monitoring.<br /> <strong>Stateful rules</strong> – Criteria for inspecting network traffic packets in the context of their traffic flow.<br /> <strong>Stateful rule groups</strong> – These define the criteria for examining a packet in the context of traffic flow, and of other traffic that’s related to the packet. The three types of stateful rule group are as follows:</p> 
<ol> 
 <li>Suricata compatible IPS rules – Defines intrusion prevention system (IPS) rules in the rule group, in Suricata compatible format. If it meets your needs, you can provide all of your stateful rules using this method.</li> 
 <li>Domain list entry – Defines a list of domain names and specifies the protocol type to inspect.</li> 
 <li>5-Tuple – Defines standard, 5-tuple criteria for examining a packet within the context of a traffic flow. You specify the source IP, source port, destination IP, destination port, and protocol, and specify the action to take for matching traffic.</li> 
</ol> 
<p><strong>HOME_NET</strong> – This is a variable used to expand the local network definition beyond the CIDR range of the VPC where you deploy Network Firewall. The HOME_NET variable is used by the Suricata engine to identify and filter traffic based on local source IPs, or “home” IPs. For the scenarios that follow, we have replaced the HOME_ NET variable with SPOKEVPC_A and SPOKEVPC_B. For further details, refer to the AWS documentation <a href="https://docs.aws.amazon.com/network-firewall/latest/developerguide/stateful-rule-groups-domain-names.html">here</a>.</p> 
<p><strong>Squid proxy modes</strong>:</p> 
<ol> 
 <li>Transparent proxy mode – This is the simplest way to configure a Squid proxy. Instances that are accessing the internet are not aware they are behind a proxy. Configuring Squid proxy in transparent mode is beneficial as users do not need to configure any specific proxy settings on the instance to access the internet.</li> 
 <li>Explicit proxy mode – This is the default mode of operation for Squid. Instances that access the internet are aware of the proxy’s presence. Spyware and worms that use the web for transmission may not be able to function because they are not aware of the proxy settings. This can reduce the spread of malicious software and prevent bandwidth from being wasted by infected systems.</li> 
</ol> 
<h2>Architecture</h2> 
<p>There are multiple <a href="https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/">deployment architectures</a> possible with AWS Network Firewall. Customers with multiple AWS accounts and VPCs currently implement Squid open-source proxies to filter their egress web traffic to the internet. These Squid instances are typically deployed from a central account. The proposed solution can be deployed in a centralized model, or in a distributed model with firewall endpoints in individual AWS accounts.</p> 
<p>The diagram that follows (figure 1) illustrates instances in spoke VPC A and spoke VPC B that require internet access through a centralized filtering solution in a Central Egress VPC. This Central VPC will contain the AWS Network Firewall endpoint, NAT Gateway (NAT GW), and Internet Gateway (internet gateway).</p> 
<p><img alt="Traffic between VPC and internet via centralized egress VPC protected by AWS Network Firewall" class="aligncenter size-full wp-image-8624" height="1247" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2021/08/24/image-v3.png" width="2627" /></p> 
<h2>Scenarios</h2> 
<h3><strong>Walk-through of three sample Squid web proxy configurations</strong></h3> 
<p>There are multiple ways to deploy Squid. In the next section, we walk through three common deployment models to filter egress internet traffic. The right model for you depends on your use case and requirements. For each example, we display a sample Squid proxy configuration and then walk-through the equivalent AWS Network Firewall configuration. We use Amazon Elastic Compute Cloud (EC2) instances with AWS Systems Manager Session Manager to validate the setup.</p> 
<h3>Scenario 1: Permit simple domain website access from all AWS resources</h3> 
<p>In this scenario, the Squid proxy is configured to allow AWS resources from spoke VPC A and spoke VPC B access to www.example.com. The sample Squid web proxy configuration is:</p> 
<p><code>acl awsnet src 10.0.0.0/15</code></p> 
<p><code>acl webaccess dstdom_regex www\.example\.com</code></p> 
<p><code>http_access allow awsnet webaccess</code><br /> <code>http_access deny all</code></p> 
<p>With AWS Network Firewall, we use its stateful rule group feature along with the domain list option. The AWS CloudFormation snippet is provided to show how to allow HTTP and HTTPS access to a specified domain:</p> 
<pre><code class="lang-yaml">  ExampleStatefulRuleGroup1:
    Type: "AWS::NetworkFirewall::RuleGroup"
    Properties:
      Capacity: "100"
      RuleGroupName: "statefulrule1"
      Type: "STATEFUL"
      Tags:
        - Key: "Name"
          Value: "allow-domain-lists"
      RuleGroup:
        RuleVariables:
          IPSets:
            HOME_NET:
              Definition:
                - "10.0.0.0/15"
        RulesSource:
          RulesSourceList:
            TargetTypes:
              - "TLS_SNI"
              - "HTTP_HOST"
            Targets:
              - "www.example.com"
            GeneratedRulesType: "ALLOWLIST"</code></pre> 
<p>To validate, run the <a href="https://curl.se/docs/manual.html">curl</a> commands from our test instance in spoke VPC A. We can access www.example.com, but not www.example.net.</p> 
<p><code>sh-4.2$ curl https://www.example.com -o /dev/null</code><br /> <code>% Total % Received % Xferd Average Speed Time Time Time Current</code><br /> <code>Dload Upload Total Spent Left Speed</code><br /> <code>100 1203 100 1203 0 0 30075 0 --:--:-- --:--:-- —:--:— 30075</code></p> 
<p><code>sh-4.2$ curl https://www.example.net -o /dev/null</code><br /> <code>% Total % Received % Xferd Average Speed Time Time Time Current</code><br /> <code>Dload Upload Total Spent Left Speed</code><br /> <code>0 0 0 0 0 0 0 0 --:--:-- 0:00:10 —:--:— 0</code></p> 
<h3>Scenario 2: Permit separate domain website access for resources in different VPC CIDRs</h3> 
<p>In this scenario, different internet domains are accessible from spoke VPC CIDRs via the Squid proxy. The allow list is configured per source CIDR, and Squid proxy evaluates the rules based upon where the request originates. This is a common way to segment into the organizational units (for example, sandbox, development, QA, production) that are associated with different IP CIDRs.</p> 
<p>In this example, we have two environments – spoke VPC A (10.0.0.0/16) and spoke VPC B (10.1.0.0/16). For resources in spoke VPC A, we have permitted access to www.example.com and for resources in spoke VPC B, we have permitted access to www.example.net.</p> 
<p><code>acl spokevpc_a src 10.0.0.0/16</code><br /> <code>acl spokevpc_b src 10.1.0.0/16</code></p> 
<p><code>acl a_webaccess dstdom_regex www\.example\.com</code><br /> <code>acl b_webaccess dstdom_regex www\.example\.net</code></p> 
<p><code>http_access allow spokevpc_a a_webaccess</code><br /> <code>http_access allow spokevpc_b b_webaccess</code><br /> <code>http_access deny all</code></p> 
<p>With AWS Network Firewall, we use the stateful rule group consisting a Suricata compatible ruleset. We use a Suricata ruleset because it provides the flexibility to create more granular IP configurations within a ruleset. For more information on using Suricata rulesets within Network Firewall, the <a href="https://aws.amazon.com/blogs/opensource/scaling-threat-prevention-on-aws-with-suricata/">Scaling threat prevention on AWS with Suricata</a> blog post may be helpful.</p> 
<p>In this example, we defined two different IPSet Rule Variables for the different source IP CIDR ranges. Here is a snippet of AWS CloudFormation to match the Squid web configuration from the preceding example.</p> 
<pre><code class="lang-yaml">
SuricataStatefulRuleGroup2:
    Type: 'AWS::NetworkFirewall::RuleGroup'
    Properties:
      RuleGroupName: "SuricataStatefulRuleGroup2"
      Type: STATEFUL
      Capacity: 100
      RuleGroup:
        RuleVariables:
          IPSets:
            SPOKEVPC_A:
              Definition:
                - "10.0.0.0/16"
        RulesSource:
          RulesString: |-
            pass tls $SPOKEVPC_A any -&gt;; any 443 (msg:"Spoke VPC A - Site Allow"; tls.sni; content:".example.com"; priority:1; rev:1;sid:110;)
            drop tcp $SPOKEVPC_A any -&gt;; any 80 (msg:"Drop established TCP:80"; flow: from_client,established; sid:114; priority:5; rev:1;)
            drop tcp $SPOKEVPC_A any -&gt;; any 443 (msg:"Drop established TCP:443"; flow: from_client,established; sid:115; priority:5; rev:1;)

SuricataStatefulRuleGroup3:
    Type: 'AWS::NetworkFirewall::RuleGroup'
    Properties:
      RuleGroupName: "SuricataStatefulRuleGroup3"
      Type: STATEFUL
      Capacity: 100
      RuleGroup:
        RuleVariables:
          IPSets:
            SPOKEVPC_B:
              Definition:
                - "10.1.0.0/16"
        RulesSource:
          RulesString: |-
            pass tls $SPOKEVPC_B any -&gt;; any 443 (msg:"Spoke VPC B - Site Allow"; tls.sni; content:".example.net"; priority:1; rev:1;sid:110;)
            drop tcp $SPOKEVPC_B any -&gt;; any 80 (msg:"Drop established TCP:80"; flow: from_client,established; sid:114; priority:5; rev:1;)
            drop tcp $SPOKEVPC_B any -&gt;; any 443 (msg:"Drop established TCP:443"; flow: from_client,established; sid:115; priority:5; rev:1;)&lt;/code&gt;</code></pre> 
<p>To validate, again run a curl command from your test instance in spoke VPC A. We see that access to www.example.com succeeds while access to www.example.net does not.</p> 
<p><code>sh-4.2$ curl https://www.example.com -o /dev/null</code><br /> <code>% Total % Received % Xferd Average Speed Time Time Time Current</code><br /> <code>Dload Upload Total Spent Left Speed</code><br /> <code>100 1203 100 1203 0 0 30075 0 --:--:-- --:--:-- —:--:— 30075</code></p> 
<p><code>sh-4.2$ curl https://www.example.net -o /dev/null</code><br /> <code>% Total % Received % Xferd Average Speed Time Time Time Current</code><br /> <code>Dload Upload Total Spent Left Speed</code><br /> <code>0 0 0 0 0 0 0 0 --:--:-- 0:00:10 —:--:— 0</code></p> 
<p>From the test instance in spoke VPC B, we see that we are able to access www.example.net but unable to access www.example.com</p> 
<p><code>sh-4.2$ curl https://www.example.com -o /dev/null</code><br /> <code>% Total % Received % Xferd Average Speed Time Time Time Current</code><br /> <code>Dload Upload Total Spent Left Speed</code><br /> <code>0 0 0 0 0 0 0 0 --:--:-- 0:00:10 —:--:— 0</code></p> 
<p><code>sh-4.2$ curl https://www.example.net -o /dev/null</code><br /> <code>% Total % Received % Xferd Average Speed Time Time Time Current</code><br /> <code>Dload Upload Total Spent Left Speed</code><br /> <code>100 39389 0 39389 0 0 68265 0 --:--:-- --:--:-- —:--:— 68265</code></p> 
<h3>Scenario 3: Permit access with wildcard domains to resources in different VPC CIDRs</h3> 
<p>In this scenario, the Squid proxy is configured to allow different internet domains based on source CIDRs as in previous case along with addition of wildcard variables in its allow list configuration. You can configure fine grained policies to permit only specific websites based on the source networks. The example that follows shows how to permit access to only Amazon Simple Storage Service (S3) buckets in US East Regions (for example, s3.us-east-1.amazonaws.com or s3.us-east-2.amazonaws.com) from resources in spoke VPC B.</p> 
<p><code>acl SPOKEVPC_A src 10.0.0.0/16</code><br /> <code>acl SPOKEVPC_B src 10.1.0.0/16</code></p> 
<p><code>acl a_webaccess dstdom_regex www\.example\.com</code><br /> <code>acl a_webaccess dstdom_regex www\.example\.net</code><br /> <code>acl b_webaccess dstdom_regex s3\.us-east.*-\d+\.amazonaws\.com</code></p> 
<p><code>http_access allow SPOKEVPC_A a_webaccess</code><br /> <code>http_access allow SPOKEVPC_B b_webaccess</code><br /> <code>http_access deny all</code></p> 
<p>Again, with AWS Network Firewall, we use the stateful rule group consisting of a Suricata compatible ruleset. We re-use the two different IP source VPC CIDRs and two separate Stateful Rule Group policies as in previous scenario. We then use the <a href="https://redmine.openinfosecfoundation.org/projects/suricata/wiki/Pcre_(Perl_Compatible_Regular_Expressions)">Perl Compatible Regular Expressions</a> (pcre) to match on the specific keywords in the domain name, as found in the Squid configuration. For reference, here is an AWS CloudFormation snippet that shows this approach:</p> 
<pre><code class="lang-yaml">SuricataStatefulRuleGroup4:
    Type: 'AWS::NetworkFirewall::RuleGroup'
    Properties:
      RuleGroupName: "SuricataStatefulRuleGroup4"
      Type: STATEFUL
      Capacity: 100
      RuleGroup:
        RuleVariables:
          IPSets:
            SPOKEVPC_A:
              Definition:
                - "10.0.0.0/16"
        RulesSource:
          RulesString: |-
            pass tls $SPOKEVPC_A any -&gt; any 443 (msg:"Spoke VPC A - Site Allow"; tls.sni; content:".example.com"; priority:1; rev:1;sid:110;)
            pass tls $SPOKEVPC_A any -&gt; any 443 (msg:"Spoke VPC A - Site Allow"; tls.sni; content:".example.net"; priority:1; rev:1;sid:111;)
            drop tcp $SPOKEVPC_A any -&gt; any 80 (msg:"Drop established TCP:80"; flow: from_client,established; sid:200; priority:5; rev:1;)
            drop tcp $SPOKEVPC_A any -&gt; any 443 (msg:"Drop established TCP:443"; flow: from_client,established; sid:201; priority:5; rev:1;)

SuricataStatefulRuleGroup5:
    Type: 'AWS::NetworkFirewall::RuleGroup'
    Properties:
      RuleGroupName: "SuricataStatefulRuleGroup5"
      Type: STATEFUL
      Capacity: 100
      RuleGroup:
        RuleVariables:
          IPSets:
            SPOKEVPC_B:
              Definition:
                - "10.1.0.0/16"
        RulesSource:
          RulesString: |-
            pass tls $SPOKEVPC_B any -&gt; any 443 (msg:"Spoke VPC B - AWS VPCEndpoint Allow"; tls.sni; content: "."; pcre:"/s3.us-.*-\d.amazonaws.com/"; priority:1; rev:1;sid:400;)
            drop tcp $SPOKEVPC_B any -&gt; any 80 (msg:"Drop established TCP:80"; flow: from_client,established; sid:500; priority:5; rev:1;)
            drop tcp $SPOKEVPC_B any -&gt; any 443 (msg:"Drop established TCP:443"; flow: from_client,established; sid:501; priority:5; rev:1;)</code></pre> 
<p>To validate, run a curl command from your test instance in spoke VPC A. You are able to successfully access www.example.com and www.example.net. Curl requests to www.example.org, however, fail.</p> 
<p><code>sh-4.2$ curl https://www.example.com -o /dev/null</code><br /> <code>% Total&nbsp;&nbsp;&nbsp; % Received % Xferd&nbsp; Average Speed&nbsp;&nbsp; Time&nbsp;&nbsp;&nbsp; Time&nbsp;&nbsp;&nbsp;&nbsp; Time&nbsp; Current</code><br /> <code>Dload&nbsp; Upload&nbsp;&nbsp; Total&nbsp;&nbsp; Spent&nbsp;&nbsp;&nbsp; Left&nbsp; Speed</code><br /> <code>100&nbsp; 2671&nbsp; 100&nbsp; 2671&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp; 62116&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 0 --:--:-- --:--:-- --:--:-- 62116</code></p> 
<p><code>sh-4.2$ curl https://www.example.net -o /dev/null</code><br /> <code>% Total&nbsp;&nbsp;&nbsp; % Received % Xferd&nbsp; Average Speed&nbsp;&nbsp; Time&nbsp;&nbsp;&nbsp; Time&nbsp;&nbsp;&nbsp;&nbsp; Time&nbsp; Current</code><br /> <code>Dload&nbsp; Upload&nbsp;&nbsp; Total&nbsp;&nbsp; Spent&nbsp;&nbsp;&nbsp; Left&nbsp; Speed</code><br /> <code>100 39389&nbsp;&nbsp;&nbsp; 0 39389&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp; 54180&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 0 --:--:-- --:--:-- --:--:-- 54105</code></p> 
<p><code>sh-4.2$ curl https://www.example.org -o /dev/null</code><br /> <code>% Total&nbsp;&nbsp;&nbsp; % Received % Xferd&nbsp; Average Speed&nbsp;&nbsp; Time&nbsp;&nbsp;&nbsp; Time&nbsp;&nbsp;&nbsp;&nbsp; Time&nbsp; Current</code><br /> <code>Dload&nbsp; Upload&nbsp;&nbsp; Total&nbsp;&nbsp; Spent&nbsp;&nbsp;&nbsp; Left&nbsp; Speed</code><br /> <code>0&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 0 --:--:--&nbsp; 0:00:10 --:--:--&nbsp;&nbsp;&nbsp;&nbsp; 0</code></p> 
<p>Run a curl from our test instance in spoke VPC B. You are able to access the two S3 endpoints in us-east-1 and us-east-2 but not to www.example.com and www.example.net. Note, a 405 response for S3 is expected as we are not passing credentials.&nbsp;<code></code></p> 
<p><code>sh-4.2$ curl -I --silent https://s3.us-west-1.amazonaws.com | grep HTTP<br /> HTTP/1.1 405 Method Not Allowed</code></p> 
<p><code>sh-4.2$ curl -I --silent https://s3.us-east-1.amazonaws.com | grep HTTP</code><br /> <code>HTTP/1.1 405 Method Not Allowed</code></p> 
<p><code>sh-4.2$ curl https://www.example.com -o /dev/null</code><br /> <code>% Total&nbsp;&nbsp;&nbsp; % Received % Xferd&nbsp; Average Speed&nbsp;&nbsp; Time&nbsp;&nbsp;&nbsp; Time&nbsp;&nbsp;&nbsp;&nbsp; Time&nbsp; Current</code><br /> <code>Dload&nbsp; Upload&nbsp;&nbsp; Total&nbsp;&nbsp; Spent&nbsp;&nbsp;&nbsp; Left&nbsp; Speed</code><br /> <code>0&nbsp; &nbsp;&nbsp;&nbsp;0&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 0 --:--:--&nbsp; 0:00:10 --:--:--&nbsp;&nbsp;&nbsp;&nbsp; 0</code><br /> <code></code><br /> <code>sh-4.2$ curl https://www.example.net -o /dev/null</code><br /> <code>% Total&nbsp;&nbsp;&nbsp; % Received % Xferd&nbsp; Average Speed&nbsp;&nbsp; Time&nbsp;&nbsp;&nbsp; Time&nbsp;&nbsp;&nbsp;&nbsp; Time&nbsp; Current</code><br /> <code>Dload&nbsp; Upload&nbsp;&nbsp; Total&nbsp;&nbsp; Spent&nbsp;&nbsp;&nbsp; Left&nbsp; Speed</code><br /> <code>0&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 0 --:--:--&nbsp; 0:00:10 --:--:--&nbsp;&nbsp;&nbsp;&nbsp; 0</code></p> 
<h2>A tool to help</h2> 
<p>We have created a simple tool that automates some of the migration tasks involved when moving from Squid proxy to AWS Network Firewall. The tool takes in a list of allow or deny lists associated with your Squid configuration and translates them to AWS Network Firewall domain-type rule groups. The tool uses S3 event trigger and Lambda for translations. The Tool repository can be found <a href="https://github.com/aws-samples/migrating-to-aws-network-firewall-from-squid-web-proxy">here</a>.</p> 
<p><em>Note :</em> This tool generates domain based rule groups with AWS Network Firewall based on the squid configuration file uploaded to a S3 bucket. The rule groups can be created for both distributed or centralized deployment models.</p> 
<h2>Cleanup</h2> 
<p>To avoid ongoing charges, delete the resources you created. Go to the AWS Management Console, identify the resources you created (the AWS Network Firewall and AWS Transit Gateway) and delete them. If you are using the tool, then also delete the S3 bucket, lambda functions, and CloudWatch log groups.</p> 
<h2>Conclusion</h2> 
<p><a href="https://aws.amazon.com/network-firewall/">AWS Network Firewall</a> provides the ability to transparently inspect traffic at scale in a variety of use cases. AWS handles the undifferentiated heavy lifting of deploying the resources, its patch management, and ensure performance at scale–so that security teams can focus less on operational burdens and more on strategic opportunities. In this post, we looked at three different scenarios for egress filtering based on domain names that use <a href="http://www.squid-cache.org/">Squid</a> as a proxy solution in AWS. We then detailed the configurations that customers use on their Squid proxy and provided an equivalent design and solution using <a href="https://aws.amazon.com/network-firewall/?">AWS Network Firewall</a>. We then provided a <a href="https://github.com/aws-samples/migrating-to-aws-network-firewall-from-squid-web-proxy">tool</a> that aids you in this migration process. We hope this post is helpful and we look forward to hearing about your migration to AWS Network Firewall.</p> 
<h2>About the Authors</h2> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="micoyne-image"><img class="wp-image-3604 size-full alignleft" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2021/06/05/download-1.jpg" /></p> 
 <h3 class="lb-h4">Shiva Vaidyanathan</h3> 
 <p style="color: #879196; font-size: 1rem;">Shiva Vaidyanathan is a Senior Cloud Infrastructure Architect at AWS. He provides technical guidance, design and lead implementation projects to customers, ensuring their success on AWS. He works towards making cloud networking simpler for everyone. Prior to joining AWS, he worked on several NSF funded research initiatives on how to perform secure computing in public cloud infrastructures. He holds a MS in Computer Science from Rutgers University and a MS in Electrical Engineering from New York University.</p> 
</div> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="micoyne-image"><img class="wp-image-3604 size-full alignleft" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2021/08/19/rash.jpeg" /></p> 
 <h3 class="lb-h4">Rashpal Kler</h3> 
 <p style="color: #879196; font-size: 1rem;">Rashpal Kler is a Senior Solutions Architect at AWS, based in Milwaukee, Wisconsin. He has a BS in Computer Science and is passionate about technology and helping customers architect and build solutions to adopt the art of the possible on AWS.</p> 
</div>
