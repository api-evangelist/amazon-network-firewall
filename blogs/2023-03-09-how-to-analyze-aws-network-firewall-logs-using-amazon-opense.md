---
title: "How to analyze AWS Network Firewall logs using Amazon OpenSearch Service – Part 2"
url: "https://aws.amazon.com/blogs/networking-and-content-delivery/how-to-analyze-aws-network-firewall-logs-using-amazon-opensearch-service-part-2/"
date: "Thu, 09 Mar 2023 18:34:25 +0000"
author: "Sagar Gandha"
feed_url: "https://aws.amazon.com/blogs/networking-and-content-delivery/tag/aws-network-firewall/feed/"
---
<p>In <a href="https://aws.amazon.com/blogs/networking-and-content-delivery/how-to-analyze-aws-network-firewall-logs-using-amazon-opensearch-service-part-1/">part 1</a> of this blog-post series, we walked you through steps to configure Amazon OpenSearch Service to receive logs from <a href="https://aws.amazon.com/network-firewall/">AWS Network Firewall</a> using <a href="https://aws.amazon.com/kinesis/data-firehose/">Amazon Kinesis Data Firehose</a>. In this part 2, we cover steps to generate test alerts, validating them and configure dashboards in <a href="https://aws.amazon.com/opensearch-service/">Amazon OpenSearch Service</a> to visualize and analyze log data.</p> 
<h3>Generate test alerts</h3> 
<ol> 
 <li>To generate Network Firewall alert logs, use <a href="https://github.com/3CORESec/testmynids.org">testmynids.org</a> which provides testing for the detection of malicious events by network intrusion detection systems (NIDS) with test files and scripts that simulate test NIDS activities.</li> 
 <li>Connect to the Web server EC2 instance created as part of the CloudFormation stack using Session Manager. To do this, go to&nbsp;the <a href="https://console.aws.amazon.com/ec2/v2/home#Instances:v=3">EC2 Instances Console</a>, select Web server instance, and select Connect. <p></p>
  <div class="wp-caption alignnone" id="attachment_15244" style="width: 1388px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-01-EC2-instance-connection.jpg"><img alt="EC2 instance connection" class="size-full wp-image-15244" height="318" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-01-EC2-instance-connection.jpg" width="1378" /></a>
   <p class="wp-caption-text" id="caption-attachment-15244">Figure 01 – EC2 instance connection</p>
  </div></li> 
 <li>Then, choose <strong>Session Manager</strong> and select Connect. This will open a browser tab with terminal access to the Web server.</li> 
 <li>Run the following commands 
  <ol> 
   <li> <pre><code class="lang-bash">sudo yum install -y nc # installing ncat&lt;br /&gt;
curl -sSL https://raw.githubusercontent.com/3CORESec/testmynids.org/master/tmNIDS -o /tmp/tmNIDS &amp;amp;&amp;amp; 
chmod +x /tmp/tmNIDS &amp;amp;&amp;amp; /tmp/tmNIDS -h&lt;br /&gt;/tmp/tmNIDS -99</code></pre> </li> 
  </ol> </li> 
</ol> 
<h3>Validation</h3> 
<ol> 
 <li>While this command is running, go to&nbsp;<a href="https://console.aws.amazon.com/vpc/home#NetworkFirewalls:">Network Firewall</a>, select Network Firewall → Monitoring, and check Stateful Received Packets, Passed Packets, and Dropped Packets. <p></p>
  <div class="wp-caption alignnone" id="attachment_15245" style="width: 2236px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-02-Monitoring-in-AWS-Network-Firewall.jpg"><img alt="Monitoring in AWS Network Firewall" class="wp-image-15245 size-full" height="704" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-02-Monitoring-in-AWS-Network-Firewall.jpg" width="2226" /></a>
   <p class="wp-caption-text" id="caption-attachment-15245">Figure 02 – Monitoring in AWS Network Firewall</p>
  </div></li> 
 <li>Moreover, look at the Monitoring of Kinesis Data Firehose delivery stream by navigating to&nbsp;<a href="https://console.aws.amazon.com/firehose/home">Amazon Kinesis delivery streams</a> → select required delivery stream → Monitoring. <p></p>
  <div class="wp-caption alignnone" id="attachment_15246" style="width: 2236px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-03-Monitoring-in-Kinesis-Data-Firehose.jpg"><img alt="Monitoring in Kinesis Data Firehose" class="wp-image-15246 size-full" height="560" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-03-Monitoring-in-Kinesis-Data-Firehose.jpg" width="2226" /></a>
   <p class="wp-caption-text" id="caption-attachment-15246">Figure 03 – Monitoring in Kinesis Data Firehose</p>
  </div></li> 
 <li>As a last step, confirm data reception to the required index at Amazon OpenSearch Dashboard → Discover. 
  <div class="wp-caption aligncenter" id="attachment_15247" style="width: 325px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-04-Amazon-OpenSearch-index-configuration.jpg"><img alt="Amazon OpenSearch index configuration" class="size-full wp-image-15247" height="373" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-04-Amazon-OpenSearch-index-configuration.jpg" width="315" /></a>
   <p class="wp-caption-text" id="caption-attachment-15247">Figure 04 – Amazon OpenSearch index configuration</p>
  </div> 
  <ul> 
   <li>There will be a prompt to create an index pattern.</li> 
   <li><a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-05.jpg"><img alt="" class="alignnone size-full wp-image-15248" height="360" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-05.jpg" width="1399" /></a> <p></p>
    <div class="wp-caption aligncenter" id="attachment_15249" style="width: 779px;">
     <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-06-Amazon-OpenSearch-index-configuration-welcome-page.jpg"><img alt="Amazon OpenSearch index configuration welcome page" class="size-full wp-image-15249" height="421" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-06-Amazon-OpenSearch-index-configuration-welcome-page.jpg" width="769" /></a>
     <p class="wp-caption-text" id="caption-attachment-15249">Figure 06 – Amazon OpenSearch index configuration welcome page</p>
    </div></li> 
   <li>Set the index pattern with an index name configured in Kinesis Data Firehose delivery stream (anf-index) and select Next step. 
    <div class="wp-caption aligncenter" id="attachment_15250" style="width: 1224px;">
     <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-07-Amazon-OpenSearch-index-pattern-configuration.jpg"><img alt="Amazon OpenSearch index pattern configuration" class="size-full wp-image-15250" height="551" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-07-Amazon-OpenSearch-index-pattern-configuration.jpg" width="1214" /></a>
     <p class="wp-caption-text" id="caption-attachment-15250">Figure 07 – Amazon OpenSearch index pattern configuration</p>
    </div> 
    <div class="wp-caption aligncenter" id="attachment_15251" style="width: 2096px;">
     <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-08.jpg"><img alt="Amazon OpenSearch Bar Chart" class="wp-image-15251 size-full" height="400" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-08.jpg" width="2086" /></a>
     <p class="wp-caption-text" id="caption-attachment-15251">Figure 08 – Amazon OpenSearch Bar Chart</p>
    </div> <p></p>
    <div class="wp-caption aligncenter" id="attachment_15252" style="width: 1224px;">
     <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-09-Amazon-OpenSearch-Discovered-Dashboard.jpg"><img alt="Amazon OpenSearch Discovered Dashboard" class="size-full wp-image-15252" height="551" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-09-Amazon-OpenSearch-Discovered-Dashboard.jpg" width="1214" /></a>
     <p class="wp-caption-text" id="caption-attachment-15252">Figure 09 – Amazon OpenSearch Discovered Dashboard</p>
    </div></li> 
   <li>Set timestamp as Time field and Create index pattern:</li> 
   <li>Once again, go to&nbsp;Amazon OpenSearch Dashboard → Discover.</li> 
  </ul> </li> 
 <li>This confirms that you’re receiving Network Firewall logs to the Amazon OpenSearch Service domain. Now you can start creating dashboards and charts on log data.</li> 
</ol> 
<h3>Creating dashboard to analyze Logs</h3> 
<p>You can create visualizations to look at different metrics, and then combine them to create a dashboard that gives you the complete analysis of logs. Here we describe how to create&nbsp;<strong>Tag Cloud </strong>and<strong> Pie Chart </strong>visualizations.</p> 
<h3>Tag Cloud visualization</h3> 
<ol> 
 <li>Log in to <strong>Amazon&nbsp;OpenSearch</strong> service, select the <strong>Visualize</strong> option from the menu, and then select <strong>Create visualization. </strong>Select the <strong>Tag Cloud </strong>visualization. 
  <div class="wp-caption aligncenter" id="attachment_15253" style="width: 779px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-10-Amazon-OpenSearch-Tag-Cloud-Visualization.jpg"><img alt="Amazon OpenSearch Tag Cloud Visualization" class="wp-image-15253 size-full" height="599" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-10-Amazon-OpenSearch-Tag-Cloud-Visualization.jpg" width="769" /></a>
   <p class="wp-caption-text" id="caption-attachment-15253">Figure 10 – Amazon OpenSearch Tag Cloud Visualization</p>
  </div> <p><strong>&nbsp;</strong></p></li> 
 <li>Select the index that was created for logs, and then configure the visualization options. In the Data section, under the Metrics option, select Tag size and leave the default Count under the Aggregation. In the Buckets, select Add -&gt; Tags from the dropdown -&gt; Significant Terms in the Aggregation dropdown. -&gt;&nbsp;event.app_proto.keyword under the Field dropdown. In the Size dropdown, enter a value based on how many words you want to see in the visualization. <p></p>
  <div class="wp-caption aligncenter" id="attachment_15254" style="width: 463px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-11-Amazon-OpenSearch-Visualization-metrics-and-buckets-configuration.jpg"><img alt="Amazon OpenSearch Visualization metrics and buckets configuration" class="wp-image-15254 size-full" height="677" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-11-Amazon-OpenSearch-Visualization-metrics-and-buckets-configuration.jpg" width="453" /></a>
   <p class="wp-caption-text" id="caption-attachment-15254">Figure 11 – Amazon OpenSearch Visualization metrics and buckets configuration</p>
  </div></li> 
 <li>In the <strong>Options</strong> section, you can change the <strong>Orientations</strong> and <strong>Font size</strong> of the words. Apply your changes by selecting <strong>Update</strong>. <p></p>
  <div class="wp-caption aligncenter" id="attachment_15255" style="width: 474px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-12-Amazon-OpenSearch-Visualization-orientation-and-font-size-configuration.jpg"><img alt="Amazon OpenSearch Visualization orientation and font size configuration" class="size-full wp-image-15255" height="305" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-12-Amazon-OpenSearch-Visualization-orientation-and-font-size-configuration.jpg" width="464" /></a>
   <p class="wp-caption-text" id="caption-attachment-15255">Figure 12 – Amazon OpenSearch Visualization orientation and font size configuration</p>
  </div></li> 
 <li>You’ll see the visualization similar to this based on the data in the index. <p></p>
  <div class="wp-caption aligncenter" id="attachment_15256" style="width: 390px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-13-Amazon-OpenSearch-Tag-Cloud-Dashboard.jpg"><img alt="Amazon OpenSearch Tag Cloud Dashboard" class="size-full wp-image-15256" height="290" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-13-Amazon-OpenSearch-Tag-Cloud-Dashboard.jpg" width="380" /></a>
   <p class="wp-caption-text" id="caption-attachment-15256">Figure 13 – Amazon OpenSearch Tag Cloud Dashboard</p>
  </div></li> 
 <li>Select <strong>Save</strong>, give a name in the <strong>Title, </strong>and give some<strong> Description </strong>of the visualization to save the visualization.</li> 
</ol> 
<h3>Pie Chart visualization</h3> 
<ol> 
 <li>Select the <strong>Visualize</strong> option from the menu and then select <strong>Create visualization</strong>. Select the <strong>Pie</strong>. <p></p>
  <div class="wp-caption aligncenter" id="attachment_15257" style="width: 774px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-14-Amazon-OpenSearch-Pie-Visualization.jpg"><img alt="Amazon OpenSearch Pie Visualization" class="size-full wp-image-15257" height="620" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-14-Amazon-OpenSearch-Pie-Visualization.jpg" width="764" /></a>
   <p class="wp-caption-text" id="caption-attachment-15257">Figure 14 – Amazon OpenSearch Pie Visualization</p>
  </div></li> 
 <li>Select the same index and then configure the visualization options. In the <strong>Data</strong> section, under the <strong>Metrics</strong> option, select <strong>Slice size, </strong>and select <strong>Sum</strong> under <strong>Aggregation </strong>dropdown and <strong>netflow.bytes</strong> under the <strong>Field</strong> dropdown. In the <strong>Buckets</strong>, select <strong>Add</strong>, select <strong>Split slices </strong>from the dropdown, select <strong>Terms</strong> in the <strong>Aggregation</strong> dropdown, and then select <strong>event.src_port</strong> under the <strong>Field</strong> dropdown. In the <strong>Size</strong> dropdown, enter a value based on how many values you want to see in the visualization. Select <strong>Add</strong> again, select <strong>Split slices </strong>from the dropdown, select <strong>Terms</strong> in the <strong>Aggregation</strong> dropdown, and then select <strong>event.dest_port</strong> under the <strong>Field</strong> dropdown. In the <strong>Size</strong> dropdown, enter a value based on how many values you want to see in the visualization. 
  <div class="wp-caption aligncenter" id="attachment_15258" style="width: 570px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-15-Amazon-OpenSearch-Visualization-metrics-configuration.jpg"><img alt="Amazon OpenSearch Visualization metrics configuration" class="size-full wp-image-15258" height="390" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-15-Amazon-OpenSearch-Visualization-metrics-configuration.jpg" width="560" /></a>
   <p class="wp-caption-text" id="caption-attachment-15258">Figure 15 – Amazon OpenSearch Visualization metrics configuration</p>
  </div> 
  <div class="wp-caption aligncenter" id="attachment_15259" style="width: 550px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-16-Amazon-OpenSearch-Visualization-buckets-configuration.jpg"><img alt="Amazon OpenSearch Visualization buckets configuration" class="size-full wp-image-15259" height="618" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-16-Amazon-OpenSearch-Visualization-buckets-configuration.jpg" width="540" /></a>
   <p class="wp-caption-text" id="caption-attachment-15259">Figure 16 – Amazon OpenSearch Visualization buckets configuration</p>
  </div> <p></p>
  <div class="wp-caption aligncenter" id="attachment_15260" style="width: 563px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-17-Amazon-OpenSearch-Visualization-additional-buckets-configuration.jpg"><img alt="Amazon OpenSearch Visualization additional buckets configuration" class="size-full wp-image-15260" height="632" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-17-Amazon-OpenSearch-Visualization-additional-buckets-configuration.jpg" width="553" /></a>
   <p class="wp-caption-text" id="caption-attachment-15260">Figure 17 – Amazon OpenSearch Visualization additional buckets configuration</p>
  </div></li> 
 <li>In the <strong>Options</strong> section, you can change the <strong>Pie settings</strong> and <strong>Label settings</strong>. Apply your changes by selecting <strong>Update</strong>. <p></p>
  <div class="wp-caption aligncenter" id="attachment_15261" style="width: 577px;">
   <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-18-Amazon-OpenSearch-Visualization-pie-and-label-settings.jpg"><img alt="Amazon OpenSearch Visualization pie and label settings" class="size-full wp-image-15261" height="730" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-18-Amazon-OpenSearch-Visualization-pie-and-label-settings.jpg" width="567" /></a>
   <p class="wp-caption-text" id="caption-attachment-15261">Figure 18 – Amazon OpenSearch Visualization pie and label settings</p>
  </div></li> 
 <li>Select <strong>Save</strong>, provide a name in the <strong>Title, </strong>and give some<strong> Description </strong>to save the visualization.</li> 
</ol> 
<h3>Dashboard</h3> 
<p>You can combine visualizations that show all of the relevant information about logs. Select <strong>Dashboard</strong> from the menu and then select <strong>Create dashboard. </strong>Select<strong> Add an existing </strong>to add visualizations to the dashboard. The panel will show the visualizations and select all of the necessary visualizations. Once done, all of the selected ones will be added to the dashboard. The size of the visualizations and some more formatting changes can be completed here to arrange the visualizations properly in the dashboard. Once done, select <strong>Save</strong> to save the dashboard and provide a <strong>Title</strong> and <strong>Description</strong>.</p> 
<p>In the following sample dashboard, there are multiple visualizations, such as Pie chart, Donut chart, Horizontal and Vertical bar charts, and Tag Cloud. These focus on different metrics such as Source and Destination by Bytes Transferred and Flow count by different dimensions. These include application protocol, Source and destination IPs, Protocol, Source and Destination ports, and TCP flags.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-19-Amazon-OpenSearch-Dashboards.jpg"><img alt="Amazon OpenSearch Dashboards" class="alignnone size-full wp-image-15262" height="815" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-19-Amazon-OpenSearch-Dashboards.jpg" width="1510" /></a></p> 
<div class="wp-caption aligncenter" id="attachment_15263" style="width: 1125px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-20.jpg"><img alt="Amazon OpenSearch Dashboards" class="size-full wp-image-15263" height="344" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2023/02/23/Figure-20.jpg" width="1115" /></a>
 <p class="wp-caption-text" id="caption-attachment-15263">Figure 20 – Amazon OpenSearch Dashboard</p>
</div> 
<h2>Clean up</h2> 
<ol> 
 <li>First, clean up Network Firewall by navigating to AWS&nbsp;<a href="https://console.aws.amazon.com/cloudformation/home#/stacks">CloudFormation Stacks</a>, select the stack that you have created earlier, and Delete.</li> 
 <li>Next,&nbsp;go to&nbsp;<a href="https://console.aws.amazon.com/vpc/home#NetworkFirewallRuleGroups:">Network Firewall rule groups</a> and delete the two Suricata Stateful rule groups that you created.</li> 
 <li>Next, delete the Kinesis Data Firehose delivery stream by navigating to&nbsp;<a href="https://console.aws.amazon.com/firehose/home#/streams">Delivery streams</a>, selecting the delivery stream that you have created, and Delete.</li> 
 <li>Then, go to&nbsp;<a href="https://console.aws.amazon.com/iamv2/home#/roles">IAM Roles</a> and delete the Service role created by&nbsp;Kinesis Data Firehose delivery stream. Find the Service role required by filtering via the&nbsp;Kinesis Data Firehose delivery stream name.</li> 
 <li>Then, go to&nbsp;<a href="https://console.aws.amazon.com/s3/buckets">S3 buckets</a> and delete the bucket that you created to store the failed data of the delivery stream.</li> 
 <li>Lastly, clean up Amazon OpenSearch Service domain by navigating to <a href="https://console.aws.amazon.com/esv3/home#opensearch/domains">Domains</a>, selecting the domain that you have created, and Delete.</li> 
</ol> 
<h2>Conclusion</h2> 
<p>Altogether, this two-part blog series demonstrated the steps involved in analyzing <a href="https://aws.amazon.com/network-firewall/">AWS Network Firewall </a>We walked through how to setup Amazon OpenSearch Service Index-specific permission for Kinesis Data Firehose Service role. Furthermore, we demonstrated how to configure rules in Network Firewall and generate test alerts. Moreover, we demonstrated how to create a dashboard and visualize different metrics in Amazon OpenSearch Service. You can also get hands-on experience with AWS Services using&nbsp;<a href="https://catalog.workshops.aws/networkfirewall/en-US/intro">Network Firewall Workshop</a> and&nbsp;<a href="https://aesworkshops.com/">Amazon OpenSearch Service Workshops</a>.</p> 
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
