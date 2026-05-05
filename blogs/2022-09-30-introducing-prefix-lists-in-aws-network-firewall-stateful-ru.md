---
title: "Introducing Prefix Lists in AWS Network Firewall Stateful Rule Groups"
url: "https://aws.amazon.com/blogs/networking-and-content-delivery/introducing-prefix-lists-in-aws-network-firewall-stateful-rule-groups/"
date: "Fri, 30 Sep 2022 18:00:20 +0000"
author: "Tyler Applebaum"
feed_url: "https://aws.amazon.com/blogs/networking-and-content-delivery/tag/aws-network-firewall/feed/"
---
<p>Previously you needed to update individual <a href="https://aws.amazon.com/network-firewall/">AWS Network Firewall</a> rules when scaling your network to add new IP addresses. The release of this new feature means that you can update the relevant prefix list, and all of the Network Firewall rule groups that reference the prefix list are automatically updated. Both customer-managed and AWS-managed prefix lists can be referenced in the stateful firewall rule. Both 5-tuple and Suricata-compatible IPS rule types support referencing prefix lists.</p> 
<h2>How prefix lists referencing works</h2> 
<p>Prefix lists let you group multiple CIDR blocks into a single object. You can choose to group together common traffic sources or destinations like remote branch offices connected to AWS via SD-WAN, or customer CIDR blocks. Then, you can easily reference these prefix lists in a stateful rule group. Whenever there’s an addition or deletion of CIDR entries from the referenced prefix list, the change is automatically propagated to the rule group and thus every network firewall using the rule group.</p> 
<h2>Configuration steps</h2> 
<p>Get started by first&nbsp;<a href="https://docs.aws.amazon.com/vpc/latest/userguide/working-with-managed-prefix-lists.html">creating a prefix list</a> using the <a href="https://aws.amazon.com/cli/">AWS Command Line Interface</a> (AWS CLI), or console. Then follow along with the following examples. Alternatively, you can use an <a href="https://docs.aws.amazon.com/vpc/latest/userguide/working-with-aws-managed-prefix-lists.html">AWS-managed prefix list</a>. If you already have the required prefix list created, then you can skip this step.</p> 
<h3>Example 1: Using a prefix list in a 5-tuple rule</h3> 
<ol> 
 <li>Navigate to the <strong>AWS Network Firewall</strong> section in the VPC management console. Choose <strong>Network Firewall rule groups</strong> and choose <strong>Create Network Firewall rule group</strong>.</li> 
 <li>Select <strong>Stateful rule group</strong> and complete the required fields. Refer to <a href="https://docs.aws.amazon.com/network-firewall/latest/developerguide/rule-group-stateful-creating.html">Creating a stateful rule group</a> for more information. Next, select <strong>5-tuple</strong> as shown in the following figure. <p></p>
  <div class="wp-caption alignnone" id="attachment_13346" style="width: 889px;">
   <img alt="Creating the Network Firewall stateful rule group" class="size-full wp-image-13346" height="849" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2022/09/15/figure-1.png" width="879" />
   <p class="wp-caption-text" id="caption-attachment-13346">Figure 1: Network Firewall rule group creation wizard</p>
  </div></li> 
 <li>Expand the <strong>IP set reference</strong> section as shown in the following figure, and choose <strong>Add another IP set reference</strong>. Give a friendly name to the IP set reference variable and select the IP set reference ID for the prefix list that you want to reference in the rule. You can define one or more IP set reference variables in this step. <p></p>
  <div class="wp-caption alignnone" id="attachment_13347" style="width: 895px;">
   <img alt="Defining the IP set reference variable" class="size-full wp-image-13347" height="658" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2022/09/15/figure-2.png" width="885" />
   <p class="wp-caption-text" id="caption-attachment-13347">Figure 2: IP set reference section</p>
  </div></li> 
 <li>Complete the <strong>Add rule</strong> section. See <a href="https://docs.aws.amazon.com/network-firewall/latest/developerguide/rule-group-stateful-creating.html">Creating a stateful rule group</a> for more information. Then, in either the source or the destination field, you can use the friendly name that you created in the previous step prefixed with the ‘<code>@</code>’ symbol. In our example, it’s <code>@branchoffices</code>, as shown in the following figure. Configure the traffic direction and the rule action as pass, drop, or alert, depending on your preferences. <p></p>
  <div class="wp-caption alignnone" id="attachment_13348" style="width: 891px;">
   <img alt="Referencing the newly-created IP set reference variable in the rule" class="size-full wp-image-13348" height="662" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2022/09/15/figure-3.png" width="881" />
   <p class="wp-caption-text" id="caption-attachment-13348">Figure 3: Add rule section</p>
  </div></li> 
 <li>Next, choose <strong>Add rule</strong> and you can see that the rule is successfully created as shown in the following figure. <p></p>
  <div class="wp-caption alignnone" id="attachment_13349" style="width: 1046px;">
   <img alt="Reviewing the stateful rule inside the rule group before final creation" class="size-full wp-image-13349" height="182" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2022/09/15/figure-4.jpg" width="1036" />
   <p class="wp-caption-text" id="caption-attachment-13349">Figure 4: Rules inside the rule group</p>
  </div></li> 
 <li>Next, choose <strong>Create stateful rule group</strong> and you can see that the rule group is successfully created.&nbsp;Once the rule group is created, the IP set reference will be visible when examining the rule group configuration as shown in the following figure. <p></p>
  <div class="wp-caption alignnone" id="attachment_13350" style="width: 889px;">
   <img alt="Viewing the IP set reference and prefix list in the stateful rule" class="size-full wp-image-13350" height="595" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2022/09/15/figure-5.png" width="879" />
   <p class="wp-caption-text" id="caption-attachment-13350">Figure 5: Rule group configuration showing the IP set reference</p>
  </div></li> 
</ol> 
<h3>Example 2: Using a prefix list in a Suricata compatible IPS rule</h3> 
<ol> 
 <li>Navigate to the <strong>AWS Network Firewall</strong> section in the VPC management console. Choose <strong>Network Firewall rule groups</strong> and choose <strong>Create Network Firewall rule group</strong>.</li> 
 <li>Select <strong>Stateful rule group</strong> and complete the required fields. See&nbsp;<a href="https://docs.aws.amazon.com/network-firewall/latest/developerguide/rule-group-stateful-creating.html">Creating a stateful rule group</a> for help. Next, select Suricata compatible IPS rules as shown in the following figure. <p></p>
  <div class="wp-caption alignnone" id="attachment_13351" style="width: 1034px;">
   <img alt="Creating a Suricata-compatible Network Firewall stateful rule" class="size-large wp-image-13351" height="442" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2022/09/15/figure-6-1024x442.png" width="1024" />
   <p class="wp-caption-text" id="caption-attachment-13351">Figure 6: Network Firewall rule group creation wizard</p>
  </div></li> 
 <li>Expand the <strong>IP set reference section </strong>as shown in the following figure, and choose <strong>Add another IP set reference</strong>. Give a friendly name to the IP set reference variable and select the IP set reference ID for the prefix list that you want to reference in the rule. You can define one or more IP set reference variables in this step. <p></p>
  <div class="wp-caption alignnone" id="attachment_13352" style="width: 889px;">
   <img alt="Creating the IP set reference in the Suricata-compatible stateful rule" class="size-full wp-image-13352" height="408" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2022/09/15/figure-7.png" width="879" />
   <p class="wp-caption-text" id="caption-attachment-13352">Figure 7: IP set reference section</p>
  </div></li> 
 <li>In Suricata compatible IPS rules section, enter the rule or rules that you created, and reference the friendly name of the IP set reference that you defined earlier using an <code>@</code> symbol. In our example, it’s <code>@Customer1Subnet</code>&nbsp;as shown in the following figure. <p></p>
  <div class="wp-caption alignnone" id="attachment_13353" style="width: 891px;">
   <img alt="Reviewing the Suricata rule string with the IP set reference variable" class="size-full wp-image-13353" height="189" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2022/09/15/figure-8.png" width="881" />
   <p class="wp-caption-text" id="caption-attachment-13353">Figure 8: Network Firewall rule group creation wizard showing the example rule</p>
  </div></li> 
 <li>Choose Create stateful rule group as shown in the following figure and you can see the rule group successfully created. <p></p>
  <div class="wp-caption alignnone" id="attachment_13354" style="width: 903px;">
   <img alt="Finalizing the stateful rule by selecting “Create stateful rule group”" class="size-full wp-image-13354" height="189" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2022/09/15/figure-9.png" width="893" />
   <p class="wp-caption-text" id="caption-attachment-13354">Figure 9: Create stateful rule group button</p>
  </div></li> 
 <li>Select the rule group that you just created and verify the IP set reference in the Rules section, as well as the IP set reference section as shown in the following figure. <p></p>
  <div class="wp-caption alignnone" id="attachment_13355" style="width: 991px;">
   <img alt="Reviewing the completed Suricata rule group" class="size-full wp-image-13355" height="506" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2022/09/15/figure-10.png" width="981" />
   <p class="wp-caption-text" id="caption-attachment-13355">Figure 10: Rule group configuration showing the IP set reference</p>
  </div></li> 
</ol> 
<h3>Considerations</h3> 
<p>Note the following considerations:</p> 
<ul> 
 <li>Prefix lists work with stateful rules. You can use prefix lists with Suricata compatible rules and 5-tuple rules to filter by source and destination IP, port, and protocol. Prefix lists work with action-based ordering (pass, drop, alert) and strict (numeric) rule ordering. You can’t use prefix lists with stateless rules or FQDN rules.</li> 
 <li>For referencing in a stateful rule, you can choose to create your own custom prefix lists or use a prefix list managed by AWS.</li> 
 <li>When referencing an IP set variable, make sure that you use the syntax <code>@prefix-list-name</code> rather than <code>$prefix-list-name</code>.</li> 
 <li>As of the writing of this post, <a href="https://docs.aws.amazon.com/whitepapers/latest/ipv6-on-aws/ipv6-security-and-monitoring-considerations.html#aws-network-firewall">Network Firewall only supports IPv4 traffic</a>. Although prefix lists can contain IPv6 entries, Network Firewall currently works with IPv4 prefix lists only. If you attempt to add an IPv6 prefix list, then an error message will be displayed in the console.</li> 
 <li>1,000 CIDRs is the default limit for the number of entries in a prefix list. This limit is adjustable in the Prefix List Service.</li> 
 <li>The ability to reference prefix lists in stateful rule groups is available now in all commercial AWS regions. <a href="https://aws.amazon.com/govcloud-us/">AWS GovCloud</a> support is coming soon.</li> 
 <li>There’s no additional cost for using Prefix Lists with <a href="https://aws.amazon.com/network-firewall/pricing/">Network Firewall</a>. Refer to the service <a href="https://docs.aws.amazon.com/network-firewall/latest/developerguide/getting-started.html">documentation</a> to get stared.</li> 
</ul> 
<h3>Conclusion</h3> 
<p>The ability to reference prefix lists in Network Firewall rule groups makes the management of groups of networks easier for various use cases. This feature will benefit organizations that wish to more tightly control their Network Firewall rules. Prefix lists can also be referenced across accounts, which makes central management of prefix lists possible. For further information about this feature, refer to the <a href="https://docs.aws.amazon.com/network-firewall/latest/developerguide/rule-groups-ip-set-references.html#rule-groups-referencing-prefix-lists">AWS Network Firewall User Guide</a>.</p> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="Akshay-Karanth-Author.jpg"><img alt="Akshay Karanth" class="alignleft wp-image-1288 size-thumbnail" height="125" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2022/09/26/Akshay-Karanth-Author.jpg" width="120" /></p> 
 <h3 class="lb-h4">Akshay Karanth</h3> 
 <p style="color: #879196; font-size: 1rem;">Akshay is a senior solutions architect at AWS. He helps digital native businesses learn, build, and grow in the AWS Cloud. Before AWS, he worked at companies such as Juniper Networks and Microsoft in various customer facing roles across networking and security domains. When not at work, Akshay enjoys hiking up a hard trail or cooking a fulfilling meal with his family.</p> 
</div> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="tapple-125.jpg"><img alt="Tyler Applebaum" class="alignleft wp-image-1288 size-thumbnail" height="125" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2022/09/26/tapple-125.jpg" width="125" /></p> 
 <h3 class="lb-h4">Tyler Applebaum</h3> 
 <p style="color: #879196; font-size: 1rem;">Tyler is a Sr. Solutions Architect in the Charlotte, NC area helping customers migrate to AWS and modernize their applications. He has previous experience as a network engineer working in healthcare and finance.</p> 
</div>
