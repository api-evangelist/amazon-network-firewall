---
title: "How to analyze AWS Network Firewall logs using Amazon OpenSearch Service – Part 1"
url: "https://aws.amazon.com/blogs/networking-and-content-delivery/how-to-analyze-aws-network-firewall-logs-using-amazon-opensearch-service-part-1/"
date: "Thu, 09 Mar 2023 18:33:18 +0000"
author: "Sagar Gandha"
feed_url: "https://aws.amazon.com/blogs/networking-and-content-delivery/tag/aws-network-firewall/feed/"
---
<p>This two-part blog series demonstrates how to build network analytics and visualizations using data available through<a href="https://aws.amazon.com/network-firewall/"> AWS Network Firewall</a> logs. Network Firewall supports&nbsp;<a href="https://aws.amazon.com/kinesis/data-firehose/">Amazon Kinesis Data Firehose</a> as one of the logging destinations, and these logs can be streamed to&nbsp;<a href="https://aws.amazon.com/opensearch-service/">Amazon OpenSearch Service</a> as a delivery destination. Network Firewall logs contain several data points, such as source IPs, destination IPs, source and destination ports, netflow bytes, protocols, etc., for traffic that are dropped or alerted based on the rules configured. These are high in volume and become difficult to analyze directly from logs. To simplify this analysis, logs can be sent to Amazon OpenSearch Service where you can index log data, visualize, and analyze using dashboards.</p> 
<p>In part 1, we walk you through steps to configure Amazon OpenSearch Service to receive logs from AWS Network Firewall using Amazon Kinesis Data Firehose.</p> 
<h3>Architecture overview</h3> 
<div class="wp-caption aligncenter" id="attachment_15203" style="width: 962px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-1-Solution-Architecture.jpg"><img alt="Solution Architecture" class="wp-image-15203 size-full" height="504" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-1-Solution-Architecture.jpg" width="952" /></a>
 <p class="wp-caption-text" id="caption-attachment-15203">Figure 1 – Solution Architecture</p>
</div> 
<p>Here is the functional flow of this architecture:</p> 
<ol> 
 <li>Network Firewall consistently inspects and monitors Network traffic to and from your VPC. Suricata Intrusion Prevention System (IPS) rules configured as a Network Firewall Stateful rule group detect threats and block attacks against known vulnerabilities, as well as create alert logs.</li> 
 <li>These logs are directly written to the Kinesis Data Firehose delivery stream through Direct PUT.</li> 
 <li>Kinesis Data Firehose transports log data to Amazon OpenSearch Service.</li> 
 <li>Amazon OpenSearch Service allows&nbsp;Kinesis Data Firehose to create and use the index through the domain level access policy and&nbsp;index-specific permission for Kinesis Data Firehose’s Service role.</li> 
 <li>Visualize and analyze Network Firewall logs in Amazon OpenSearch Service using Amazon OpenSearch Service Dashboards.</li> 
</ol> 
<p>The following steps configure this architecture in your AWS account.</p> 
<h3>Prepare Amazon OpenSearch Service</h3> 
<ol> 
 <li>First, choose the region of your choice where Amazon OpenSearch Service is supported, and create a new Amazon OpenSearch Service domain through → <a href="https://console.aws.amazon.com/esv3/home">Amazon OpenSearch Service</a>→ Create domain. If you prefer to use an existing domain, then you can skip this section.</li> 
 <li>Set a domain name. In this example, I am using the domain name <strong>anf-logs-domain</strong>.</li> 
 <li>For this example, I am using the deployment type <strong>Development and testing, latest version, 1-AZ, t3.medium.search and number of nodes=1</strong>. Note that t3.medium.search instance incurs costs. Refer <a href="https://aws.amazon.com/opensearch-service/pricing/">here</a> for the pricing details of search instances.</li> 
 <li>You can choose the Deployment type as required for your use case. Refer to <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/bp.html">Operational best practices for Amazon OpenSearch Service</a> for best practices regarding operating Amazon OpenSearch Service domains and general guidelines that apply to many use cases. 
  <div class="wp-caption aligncenter" id="attachment_15204" style="width: 1658px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-2-Amazon-OpenSearch-domain-creation.jpg"><img alt="Amazon OpenSearch domain creation" class="wp-image-15204 size-full" height="1962" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-2-Amazon-OpenSearch-domain-creation.jpg" width="1648" /></a>
   <p class="wp-caption-text" id="caption-attachment-15204">Figure 2 – Amazon OpenSearch domain creation</p>
  </div> 
  <div class="wp-caption alignnone" id="attachment_15205" style="width: 1658px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-3-Amazon-OpenSearch-domain-creation-nodes-configuration.jpg"><img alt="Amazon OpenSearch domain creation - nodes configuration" class="size-full wp-image-15205" height="1750" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-3-Amazon-OpenSearch-domain-creation-nodes-configuration.jpg" width="1648" /></a>
   <p class="wp-caption-text" id="caption-attachment-15205">Figure 3 – Amazon OpenSearch domain creation – nodes configuration</p>
  </div> <p>a. Select Network as <strong>Public access. </strong>Make sure that fine-grain access control is enabled, select <strong>create master user,</strong> and set the master username and password.&nbsp;For testing and development purposes, we’re configuring the domain with Public Access and leveraging on Domain Access policies for IP restrictions and Basic Authentication using fine-grain access control. For production use cases, consider using VPC access. Select <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/fgac.html">here</a> to learn more about fine-grain access control in Amazon OpenSearch Service.</p> <p></p>
  <div class="wp-caption alignnone" id="attachment_15206" style="width: 1642px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-4-Amazon-OpenSearch-domain-creation-FGAC-configuration.jpg"><img alt="Amazon OpenSearch domain creation - FGAC configuration" class="size-full wp-image-15206" height="1666" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-4-Amazon-OpenSearch-domain-creation-FGAC-configuration.jpg" width="1632" /></a>
   <p class="wp-caption-text" id="caption-attachment-15206">Figure 4 – Amazon OpenSearch domain creation – FGAC configuration</p>
  </div></li> 
 <li>You can use IP-based policies to restrict access to a domain for IP addresses or CIDR blocks. Configure Source IPs with IP ranges from which Amazon OpenSearch Service (for example, Dashboard URL) access will be allowed.</li> 
 <li>You can also authenticate your Amazon OpenSearch Service Dashboards using <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/saml.html">SAML authentication</a> or <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/cognito-auth.html">Amazon Cognito authentication</a>. We’re skipping these authentication methods for this testing and development purpose.</li> 
 <li>To do this, under <strong>Access policy</strong>, choose&nbsp;<strong>Configure domain level access policy</strong> → JSON, and update the policy with the following sample policy. This access policy allows access only to the required <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ac.html#ac-types-ip">source IP ranges</a>.&nbsp;Remember to update the following sample policy before using it: 
  <ul> 
   <li>Update aws:SourceIP with the IP address ranges from which the Amazon OpenSearch Service Dashboard URL will be accessed.</li> 
   <li>Change the region at the ARN of Resource with the region of the Amazon OpenSearch Service domain.</li> 
   <li>Change the account ID at the ARN of Resource with the account ID of the Amazon OpenSearch Service domain</li> 
  </ul> </li> 
 <li>For more information on access policies, see <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ac.html">Identity and Access Management in Amazon OpenSearch Service</a>.</li> 
</ol> 
<pre><code class="lang-json">{ 
   "Version": "2012-10-17", 
   "Statement": [ 
      { 
         "Effect": "Allow", 
         "Principal": { 
             "AWS": "*" 
         },
         "Action": [
              "es:ESHttp*" 
          ], 
          "Condition": {
              "IpAddress": { 
                  "aws:SourceIp": [ 
                     "203.0.113.0/24", 
                     "198.51.100.0/24" 
                  ] 
              } 
           }, 
           "Resource": "arn:aws:es:us-east-2:111122223333:domain/anf-logs-domain/*" 
        } 
    ]
 }</code></pre> 
<div class="wp-caption aligncenter" id="attachment_15207" style="width: 823px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-5-Amazon-OpenSearch-domain-creation-access-policy-configuration.jpg"><img alt="Amazon OpenSearch domain creation - access policy configuration" class="size-full wp-image-15207" height="813" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-5-Amazon-OpenSearch-domain-creation-access-policy-configuration.jpg" width="813" /></a>
 <p class="wp-caption-text" id="caption-attachment-15207">Figure 5 – Amazon OpenSearch domain creation – access policy configuration</p>
</div> 
<p>9. Leave the rest of the configuration as default and select Create. Domain creation will take 15-20 minutes. Proceed to the next step after the domain becomes available.</p> 
<p>10. Test access to the Amazon OpenSearch Service Dashboard URL. You can find this URL in the General information of the domain. Log in using the master username and password that you configured.</p> 
<h3>Prepare Kinesis Data Firehose</h3> 
<ol> 
 <li>As a next step, create a <a href="https://console.aws.amazon.com/firehose/home">Kinesis Data Firehose delivery stream</a> to transport data from Network&nbsp;Firewall to Amazon OpenSearch Service. This delivery stream should be in the same account and region where&nbsp;Network&nbsp;Firewall will be.</li> 
 <li>Set the source as <strong>Direct PUT </strong>which allows applications to write directly to the delivery stream. Set the destination as Amazon OpenSearch Service.</li> 
 <li>Change the Delivery stream name if required. Under <strong>Transform records</strong>, set <strong>Data transformation</strong> to <strong>Disabled</strong>. <p></p>
  <div class="wp-caption aligncenter" id="attachment_15208" style="width: 821px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-6-Amazon-Kinesis-Delivery-stream-configuration.jpg"><img alt="Amazon Kinesis Delivery stream configuration" class="wp-image-15208 size-full" height="956" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-6-Amazon-Kinesis-Delivery-stream-configuration.jpg" width="811" /></a>
   <p class="wp-caption-text" id="caption-attachment-15208">Figure 6 – Amazon Kinesis Delivery stream configuration</p>
  </div></li> 
 <li>Under <strong>Destination settings,</strong> browse and choose the Amazon OpenSearch Service domain. I am choosing <strong>anf-logs-domain</strong></li> 
 <li>Set the required name for the index that will be created in the Amazon OpenSearch Service domain. I am specifying the index name as&nbsp;<strong>anf-index</strong>. This is the index at which both the alert and logs of the Network&nbsp;Firewall will be organized. You may choose to use a different index for different logs. If so, then you must configure the log destinations at&nbsp;Network&nbsp;Firewall to different Kinesis Data Firehose delivery streams. <p></p>
  <div class="wp-caption aligncenter" id="attachment_15209" style="width: 822px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-7-Amazon-Kinesis-Destination-settings.jpg"><img alt="Amazon Kinesis Destination settings" class="wp-image-15209 size-full" height="312" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-7-Amazon-Kinesis-Destination-settings.jpg" width="812" /></a>
   <p class="wp-caption-text" id="caption-attachment-15209">Figure 7 – Amazon Kinesis Destination settings</p>
  </div></li> 
 <li>Under Backup settings, create/choose S3 bucket to collect all failed data. <p></p>
  <div class="wp-caption aligncenter" id="attachment_15210" style="width: 822px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-8-Amazon-Kinesis-Backup-settings.jpg"><img alt="Amazon Kinesis - Backup settings" class="wp-image-15210 size-full" height="312" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-8-Amazon-Kinesis-Backup-settings.jpg" width="812" /></a>
   <p class="wp-caption-text" id="caption-attachment-15210">Figure 8 – Amazon Kinesis – Backup settings</p>
  </div></li> 
 <li>Leave the rest of the configuration as default and select <strong>Create delivery stream</strong>.</li> 
 <li>Once the delivery stream is active, select the delivery stream name and go to <strong>Configuration → Permissions</strong>. Here, you find the role that Kinesis Data Firehose uses for all of the delivery stream needs. Select the IAM role and copy the ARN from its summary. This IAM role should be allowed to create and use the index on the Amazon OpenSearch Service domain side.</li> 
</ol> 
<h4>Amazon OpenSearch Service Index-specific permission for the Service role</h4> 
<ol> 
 <li>For Kinesis Data Firehose to deliver data to an index in Amazon OpenSearch Service, the Service role of the Kinesis Data Firehose delivery stream must be allowed access through index-specific permission. To do so, log in to Amazon OpenSearch Service Dashboard and go to Security. <p></p>
  <div class="wp-caption aligncenter" id="attachment_15211" style="width: 340px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-9-Amazon-OpenSearch-Security-configuration.jpg"><img alt="Amazon OpenSearch Security configuration" class="wp-image-15211 size-full" height="731" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-9-Amazon-OpenSearch-Security-configuration.jpg" width="330" /></a>
   <p class="wp-caption-text" id="caption-attachment-15211">Figure 9 – Amazon OpenSearch Security configuration</p>
  </div></li> 
 <li>Create a role for access only to an index which will be used by&nbsp;Kinesis Data Firehose.</li> 
 <li>Create a new role under Security → Roles → Create role.</li> 
 <li>Give a name for the role and set the cluster permission to indices_all.</li> 
 <li>Then, set the index permission with the index name configured in Kinesis Data Firehose delivery stream (anf-index*) as pattern. Furthermore, set the permissions as indices_all. <p></p>
  <div class="wp-caption aligncenter" id="attachment_15212" style="width: 947px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-10-Amazon-OpenSearch-Role-creation.jpg"><img alt="Amazon OpenSearch Role creation" class="wp-image-15212 size-full" height="939" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-10-Amazon-OpenSearch-Role-creation.jpg" width="937" /></a>
   <p class="wp-caption-text" id="caption-attachment-15212">Figure 10 – Amazon OpenSearch Role creation</p>
  </div></li> 
 <li>For this example, I am allowing permission to all tenants. You can restrict to required tenants. <p></p>
  <div class="wp-caption aligncenter" id="attachment_15213" style="width: 947px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-11-Amazon-OpenSearch-Tenant-permissions-for-the-role.jpg"><img alt="Amazon OpenSearch Tenant permissions for the role" class="wp-image-15213 size-full" height="246" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-11-Amazon-OpenSearch-Tenant-permissions-for-the-role.jpg" width="937" /></a>
   <p class="wp-caption-text" id="caption-attachment-15213">Figure 11 – Amazon OpenSearch Tenant permissions for the role</p>
  </div></li> 
 <li>Select create role. Then, map this role to Kinesis Data Firehose Service role by selecting Mapped users → Manage mapping and set Backend role with the ARN of the Kinesis Data Firehose Service role copied earlier. <p></p>
  <div class="wp-caption aligncenter" id="attachment_15214" style="width: 947px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-12-Amazon-OpenSearch-backend-roles-mapping.jpg"><img alt="Amazon OpenSearch backend roles mapping" class="wp-image-15214 size-full" height="646" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-12-Amazon-OpenSearch-backend-roles-mapping.jpg" width="937" /></a>
   <p class="wp-caption-text" id="caption-attachment-15214">Figure 12 – Amazon OpenSearch backend roles mapping</p>
  </div></li> 
 <li>Test the Kinesis Data Firehose delivery stream by following the instructions in&nbsp;<a href="https://docs.aws.amazon.com/firehose/latest/dev/test-drive-firehose.html#test-drive-destination-elasticsearch">Test Using OpenSearch Service as the Destination</a>.</li> 
</ol> 
<h3>Prepare Network Firewall</h3> 
<ol> 
 <li>As next step, create AWS Network Firewall using AWS CloudFormation template. Click&nbsp;<a href="https://github.com/aws-samples/aws-network-firewall-demo/archive/refs/heads/main.zip">here&nbsp;</a>to download and extract the zip file locally.</li> 
 <li>Make sure that you’re in the required account and region, and then <a href="https://console.aws.amazon.com/cloudformation/home#/stacks/create/template">Create&nbsp;AWS CloudFormation stack</a>.</li> 
 <li>Choose <strong>Upload a template file</strong> and choose the&nbsp;<strong>AWS-Network-Firewall-demo.yaml </strong>that you extracted locally.</li> 
 <li>Select Next and set a name for the stack. Leave the rest as default until the Review page.</li> 
 <li>At the bottom of the Review page, select the check box <strong>I acknowledge that AWS CloudFormation might create IAM resources with custom names </strong>and select&nbsp;Create stack.</li> 
 <li>This stack will deploy resources as mentioned in the following architecture:</li> 
</ol> 
<div class="wp-caption aligncenter" id="attachment_15215" style="width: 884px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-13-AWS-Network-Firewall-configuration.jpg"><img alt="AWS Network Firewall configuration" class="size-full wp-image-15215" height="720" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-13-AWS-Network-Firewall-configuration.jpg" width="874" /></a>
 <p class="wp-caption-text" id="caption-attachment-15215">Figure 13 – AWS Network Firewall configuration</p>
</div> 
<ul> 
 <li>All inbound traffic to the Protected subnet will be routed through Network Firewall endpoint (vpce-id) using Internet Gateway Route Table, where all Stateful and Stateless Firewall rules are applied for decision on traffic flow.</li> 
 <li>All outbound traffic from Protected subnet to public will be routed through Network Firewall endpoint (vpce-id) using Protected Subnet Route Table, and sent to Internet Gateway based on the outcome of rules validation.</li> 
</ul> 
<h4>Change log destination to Kinesis Data Firehose</h4> 
<ol> 
 <li>Network Firewall created through CloudFormation stack sends alert logs to . This destination must be changed to Kinesis Data Firehose.</li> 
 <li>Get the name of the Kinesis Data Firehose delivery stream (PUT-OPS-XXXXX) created earlier.</li> 
 <li>Go to&nbsp;<a href="https://console.aws.amazon.com/vpc/home#NetworkFirewalls:">Network Firewall</a>, select Network Firewall → Firewall details → Logging → Edit.</li> 
 <li>Uncheck Alert under log type and select Save. This is to have a clean slate. Repeat the same step and select both Alert and Flow log types. Choose destinations for both log types to Kinesis Data Firehose and paste the name of the Kinesis Data Firehose delivery stream here. Select Save and confirm the log destination as shown in the following: <p></p>
  <div class="wp-caption aligncenter" id="attachment_15216" style="width: 1775px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-14-AWS-Network-Firewall-log-destination-configuration.jpg"><img alt="AWS Network Firewall log destination configuration" class="wp-image-15216 size-full" height="194" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-14-AWS-Network-Firewall-log-destination-configuration.jpg" width="1765" /></a>
   <p class="wp-caption-text" id="caption-attachment-15216">Figure 14 – AWS Network Firewall log destination configuration</p>
  </div></li> 
</ol> 
<h4>Configure rules</h4> 
<ol> 
 <li>Network Firewall supports Intrusion Prevention System (IPS), which actively inspects traffic flow with real-time network. Moreover, it protects the application layer from vulnerability exploits and brute force attacks using signature-based detection.</li> 
 <li>Network Firewall Stateful rule group uses Suricata IPS rules engine. Download&nbsp;Suricata emerging user agent rules from <a href="https://rules.emergingthreats.net/open/suricata-5.0/rules/emerging-user_agents.rules">here</a>,and create Stateful rule group. These rules will alert on different types of ET user agent activities.</li> 
 <li>To do this, go to&nbsp;<a href="https://console.aws.amazon.com/vpc/home#NetworkFirewallRuleGroups:">Network Firewall rule groups</a> and Create Network Firewall rule group.</li> 
 <li>Set Rule group type to&nbsp;<strong>Stateful rule group</strong>, specify a name (<strong>EmergingUserAgentRuleGroup</strong>), and set Capacity to 500.&nbsp;Capacity can’t be exceeded or modified once set. Therefore, configure with enough room for your rule group to grow. <p></p>
  <div class="wp-caption alignnone" id="attachment_15217" style="width: 2668px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-15-AWS-Network-Firewall-Stateful-rule-group-configuration.jpg"><img alt="AWS Network Firewall Stateful rule group configuration" class="wp-image-15217 size-full" height="1122" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-15-AWS-Network-Firewall-Stateful-rule-group-configuration.jpg" width="2658" /></a>
   <p class="wp-caption-text" id="caption-attachment-15217">Figure 15 – AWS Network Firewall Stateful rule group configuration</p>
  </div></li> 
 <li>Choose <strong>Suricata compatible IPS rules</strong> as Stateful rule group option. <p></p>
  <div class="wp-caption alignnone" id="attachment_15218" style="width: 2668px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-16-AWS-Network-Firewall-Suricata-IPS-rules-selection.jpg"><img alt="AWS Network Firewall Suricata IPS rules selection" class="wp-image-15218 size-full" height="442" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-16-AWS-Network-Firewall-Suricata-IPS-rules-selection.jpg" width="2658" /></a>
   <p class="wp-caption-text" id="caption-attachment-15218">Figure 16 – AWS Network Firewall Suricata IPS rules selection</p>
  </div></li> 
 <li>Copy the content of the downloaded Suricata emerging user agent rules to the Suricata compatible IPS rules section. <p></p>
  <div class="wp-caption alignnone" id="attachment_15219" style="width: 2344px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-17-AWS-Network-Firewall-Suricata-IPS-rules-configuration.jpg"><img alt="AWS Network Firewall Suricata IPS rules configuration" class="size-full wp-image-15219" height="442" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-17-AWS-Network-Firewall-Suricata-IPS-rules-configuration.jpg" width="2334" /></a>
   <p class="wp-caption-text" id="caption-attachment-15219">Figure 17 – AWS Network Firewall Suricata IPS rules configuration</p>
  </div></li> 
 <li>Leave the rest of the configurations as default and select Create Stateful rule group.</li> 
 <li>Similarly, create another Suricata Stateful rule group <strong>TestmynidsDropRuleGroup</strong>with Capacity=50 as mentioned in the following. This rule will drop packets and generate alerts when the content of the packet has a string value uid=0|28|root|29| and the traffic is classified as bad-unknown. You can test this rule from the Web server <a href="https://aws.amazon.com/ec2/">Amazon Elastic Compute Cloud (Amazon EC2)</a> instance using <code>curl --max-time 5</code>&nbsp;<a href="http://testmynids.org/uid/index.html">http://testmynids.org/uid/index.html</a>. 
  <div class="wp-caption alignnone" id="attachment_15220" style="width: 2668px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-18-AWS-Network-Firewall-TestmynidsDropRuleGroup-Stateful-rule-group-configuration.jpg"><img alt="AWS Network Firewall TestmynidsDropRuleGroup Stateful rule group configuration" class="wp-image-15220 size-full" height="1122" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-18-AWS-Network-Firewall-TestmynidsDropRuleGroup-Stateful-rule-group-configuration.jpg" width="2658" /></a>
   <p class="wp-caption-text" id="caption-attachment-15220">Figure 18 – AWS Network Firewall TestmynidsDropRuleGroup Stateful rule group configuration</p>
  </div> 
  <div class="wp-caption alignnone" id="attachment_15221" style="width: 2668px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-19-AWS-Network-Firewall-TestmynidsDropRuleGroup-Suricata-IPS-rules-selection.jpg"><img alt="AWS Network Firewall TestmynidsDropRuleGroup Suricata IPS rules selection" class="wp-image-15221 size-full" height="442" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-19-AWS-Network-Firewall-TestmynidsDropRuleGroup-Suricata-IPS-rules-selection.jpg" width="2658" /></a>
   <p class="wp-caption-text" id="caption-attachment-15221">Figure 19 – AWS Network Firewall TestmynidsDropRuleGroup Suricata IPS rules selection</p>
  </div> <p></p>
  <div class="wp-caption alignnone" id="attachment_15222" style="width: 2676px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-20-AWS-Network-Firewall-TestmynidsDropRuleGroup-Suricata-IPS-rules-configuration.jpg"><img alt="AWS Network Firewall TestmynidsDropRuleGroup Suricata IPS rules configuration" class="wp-image-15222 size-full" height="446" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/20/Figure-20-AWS-Network-Firewall-TestmynidsDropRuleGroup-Suricata-IPS-rules-configuration.jpg" width="2666" /></a>
   <p class="wp-caption-text" id="caption-attachment-15222">Figure 20 – AWS Network Firewall TestmynidsDropRuleGroup Suricata IPS rules configuration</p>
  </div></li> 
 <li><code>drop ip any any -&gt; any any (msg:"GPL ATTACK_RESPONSE id check returned root"; content:"uid=0|28|root|29|"; classtype:bad-unknown; sid:2100498; rev:7; metadata:created_at 2010_09_23, updated_at 2010_09_23;)</code></li> 
 <li>Go to&nbsp;<a href="https://console.aws.amazon.com/vpc/home#NetworkFirewallPolicies:">Firewall policies</a> and select DemoFirewallPolicy.</li> 
 <li>Scroll down to Stateful rule&nbsp;groups → Select StatefulRuleGroup → Actions → Dissociate from policy.</li> 
 <li>Then, under the same Stateful rule groups (of DemoFirewallPolicy) → Actions → Add unmanaged Stateful rule group, and add both EmergingUserAgentRuleGroup and TestmynidsDropRuleGroup.</li> 
</ol> 
<h2>Conclusion</h2> 
<p>In this blog post, we demonstrated steps involved to send <a href="https://aws.amazon.com/network-firewall/">AWS Network Firewall</a> logs to <a href="https://aws.amazon.com/opensearch-service/">Amazon OpenSearch Service</a> using <a href="https://aws.amazon.com/kinesis/data-firehose/">Kinesis Data Firehose</a>. We walked through how to setup Amazon OpenSearch Service Index-specific permission for Kinesis Data Firehose Service role. Furthermore, we demonstrated how to configure rules in Network Firewall.</p> 
<p>In <a href="https://aws.amazon.com/blogs/networking-and-content-delivery/how-to-analyze-aws-network-firewall-logs-using-amazon-opensearch-service-part-2/">part 2</a> of this blog-post series, we cover steps to generate test alerts, validating them and configure dashboards in Amazon OpenSearch Service to visualize and analyze log data.</p> 
<p><strong>About the authors:</strong></p> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="praksri-125x125-2-1.jpeg"><img class="alignleft size-full wp-image-5363" height="125" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/24/praksri-125x125-2-1.jpeg" width="125" /></p> 
 <h3 class="lb-h4">Prakash Srinivasan</h3> 
 <p>Prakash is a Solutions Architect with Amazon Web Services. He is a passionate builder and helps customers to modernize their applications and accelerate their Cloud journey to get the best out of Cloud for their business. In his spare time, he enjoys watching movies and spend more time with family. He is based out of Denver, Colorado and you can connect with him on Linkedin at linkedin.com/in/prakash-s</p> 
</div> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="Sagar-Blog-Photo.jpeg"><img class="alignleft size-full wp-image-5363" height="125" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/24/Sagar-Blog-Photo.jpeg" width="125" /></p> 
 <h3 class="lb-h4">Sagar Gandha</h3> 
 <p>Sagar is an experienced Sr. Technical Account Manager, adept at assisting large customers in the Enterprise Support. He offers expert guidance on best practices, facilitates access to subject matter experts, and delivers actionable insights on optimizing AWS spend, workloads, and events. When not at work, Sagar loves spending quality time with his family (wife Anitha and son Adrit) trying out new eateries, watching movies, and socializing with friends.</p> 
</div>
