---
title: "Implementing ingress geo-restriction with AWS to reduce attack surface"
url: "https://aws.amazon.com/blogs/networking-and-content-delivery/implementing-ingress-geo-restriction-with-aws-to-reduce-attack-surface/"
date: "Mon, 05 Jan 2026 19:35:21 +0000"
author: "Rahi Patel"
feed_url: "https://aws.amazon.com/blogs/networking-and-content-delivery/tag/aws-network-firewall/feed/"
---
<p>Geo-restriction has evolved from a nice-to-have feature to a critical security control. As cyber threats become increasingly sophisticated and geographically concentrated, organizations need precise tools to control where their traffic originates—whether for security, compliance, or cost optimization. These controls integrate seamlessly into zero-trust security models, making sure that access requests originate from expected locations and adding a crucial verification layer to authentication processes.</p> 
<p>The security imperative has become even more pressing as certain geographic regions consistently generate disproportionately high levels of malicious traffic, such as sophisticated automated attacks and fraud attempts. When legitimate business rarely originates from these high-risk regions, geographic restrictions offer a powerful way to reduce attack surfaces and focus security resources on protecting genuine user traffic.</p> 
<p>In this post, we dive deep into how each Amazon Web Services (AWS) services like <a href="https://aws.amazon.com/cloudfront/">Amazon CloudFront</a>, <a href="https://aws.amazon.com/waf/">AWS WAF</a>, <a href="https://aws.amazon.com/network-firewall/">AWS Network Firewall</a> and <a href="https://aws.amazon.com/route53/">Amazon Route 53</a> effectively handle geo-restriction, analyze their key differences, and provide guidance for choosing the optimal solution for your specific needs. We demonstrate when to use each service, and how to combine them for a defense-in-depth approach, so that you can implement the most effective solution for your architecture.</p> 
<p>All AWS services covered in this post use <a href="https://www.maxmind.com/en/home">MaxMind GeoIP</a> databases to provide accurate IP-to-location mapping at country and regional levels.</p> 
<h2>Option 1: CloudFront</h2> 
<p>CloudFront is a content delivery network (CDN) that securely delivers content through a global network of edge locations. CloudFront provides several approaches for implementing geo-restriction: a feature for country-level blocking at no added cost, integration capabilities with third-party geolocation services, and edge functions for programmatic control. In this section we examine each approach in detail.</p> 
<h3>1) Built-in geolocation feature</h3> 
<p>The CloudFront <a href="https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/georestrictions.html">built-in geographic restriction feature</a> provides clear country-level access control with zero added costs beyond standard CloudFront pricing. CloudFront geographic restrictions apply uniformly across the entire distribution, meaning all cache behaviors share the same geographic rules. These restrictions cannot be configured differently for individual paths or behaviors within the same distribution.</p> 
<p>The feature operates on an exclusive basis: allowlists automatically block all non-specified countries, while blocklists allow all countries except those specified.</p> 
<p>Geographic restrictions set directly in CloudFront take precedence over any AWS WAF rules and edge functions (<a href="https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-functions.html">CloudFront Functions</a> and <a href="https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-at-the-edge.html">Lambda@Edge</a>). This precedence order makes sure that geographic restrictions act as the first line of defense in your security stack. Any traffic that passes the CloudFront geo-restriction filter is evaluated by AWS WAF rules and edge functions. Log attributes such as<code>"x-edge-detailed-result-type": "ClientGeoBlocked"</code>and<code>"sc-status": "403"</code>forbidden clearly indicate blocked requests. <a href="https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/real-time-logs.html">More log </a>attributes such as<code>"c-ip"</code>,<code>“x-forwarded-for”</code>and<code>"c-country"</code>offer deeper insights into the origin of requests.</p> 
<p>The following is a sample CloudFront access log showing a geo-blocked request from the United States:</p> 
<div class="hide-language"> 
 <pre><code class="lang-json">{
"date": "2025-08-22",
"time": "02:50:36",
"x-edge-location": "DFW57-P6",
"c-ip": "1234:1234:1f23:b123:1234:A2b3:9a1z:g123",
"cs-method": "GET",
"cs(Host)": "d1eXXXXXXXmm.cloudfront.net",
"cs-uri-stem": "/",
"sc-status": "403",
"cs(User-Agent)": "curl/8.7.1",
"x-edge-result-type": "Error",
"x-edge-request-id": "xZTYxiZQeaE4W2qZQggfLsku2smTCxR7JG3Y1BJhzPp_PAxcpXX==",
"x-host-header": "d1eXXXXXXXmm.cloudfront.net",
"cs-protocol": "http",
"x-edge-response-result-type": "Error",
"x-edge-detailed-result-type": "ClientGeoBlocked",
"sc-content-type": "text/html",
"c-country": "US"
}</code></pre> 
</div> 
<h3>2) Third-party geolocation service</h3> 
<p><a href="https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/georestrictions.html#georestrictions-geolocation-service">Third-party geolocation services</a> provide an alternative approach to implementing geographic controls beyond the CloudFront country-level restrictions. This approach can be particularly relevant for organizations that need to implement complex geographic rules that follow non-standard boundaries, or when they need different rules for specific content paths within the same distribution. Organizations might choose this path when they already use external geolocation services as part of their broader technology stack or have specific requirements that align well with third-party solutions.</p> 
<p>The implementation works by integrating your application with a third-party geolocation service’s API. When a user requests content, your application captures the client’s IP address and sends it to the geolocation service. The service returns detailed geographic data, including city, postal code, coordinates, and sometimes more context such as ISP information or connection type. Your application can use this enriched location data to make access control decisions.</p> 
<h3>3) Edge functions</h3> 
<p>Edge functions provide programmatic geo-restriction capabilities through two options: CloudFront Functions for lightweight, high-performance scenarios, and Lambda@Edge for complex logic needing more computational power. These functions enable you to implement sophisticated geographic access controls directly at CloudFront edge locations, allowing for granular blocking based on various location attributes and URL pattern matching. To use geolocation data in edge functions, you must configure either an <a href="https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/controlling-origin-requests.html">Origin Request Policy</a> or <a href="https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/controlling-the-cache-key.html">Cache Policy</a> that includes the desired viewer location headers.</p> 
<p>CloudFront Functions run with sub-millisecond latency directly at CloudFront edge locations, making them ideal for high-volume, low-latency geolocation decisions. These functions can be triggered during viewer request or viewer response events. They have access to geolocation and device data during viewer events, so they can handle millions of requests per second, making them highly scalable for high-traffic applications. Blocked requests show<code>"sc-status": "403"</code>and<code>"x-edge-detailed-result-type": "FunctionGeneratedResponse"</code>in CloudFront logs.</p> 
<p>Lambda@Edge provides another option for implementing geo-restriction logic, though with some important considerations. Although it can be triggered at both viewer and origin events, geolocation data is only available during origin requests and responses. This consideration, in combination with the fact that origin calls only occur during cache misses, makes it less ideal for primary geo-restriction implementation. However, if you already use Lambda@Edge functions in your architecture, then adding geographic filtering logic might be convenient. Blocked requests show<code>"sc-status": "403"</code>and<code>"x-edge-detailed-result-type": "MissGeneratedResponse"</code>in logs. Lambda@Edge functions are more computationally powerful but also <a href="https://aws.amazon.com/cloudfront/pricing/">more expensive</a> than CloudFront Functions.</p> 
<h2>Option 2: AWS WAF</h2> 
<p>AWS WAF is a web application firewall that helps secure your web applications and APIs by blocking requests before they reach your servers.</p> 
<p>AWS WAF determines the client location by examining the source IP address from either the direct client connection or X-Forwarded-For (XFF) headers when behind proxies or CDNs. When examining X-Forwarded-For (XFF) headers, AWS WAF examines the first IP address in the header.</p> 
<p>The <a href="https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-geo-match.html">geo match</a> statement within AWS WAF automatically enriches all incoming web requests with geographical context by applying labels that indicate both the country and region of origin. These labels follow a consistent format:<code>"clientip:geo:country:&lt;ISO country code&gt;"</code>and<code>"clientip:geo:region:&lt;ISO region code&gt;"</code>for countries and regions respectively. Importantly, these labels are injected whenever a geo match rule evaluates a request, regardless of whether the rule finds a match or not. This consistent labeling serves two valuable purposes: it enables the use of these geographic identifiers in downstream rules for more complex access control logic, and it provides rich data for log analysis to understand traffic patterns and their geographic origins.</p> 
<p>AWS WAF geo match rules support Allow, Block, Count, Challenge, and CAPTCHA actions, and it can combine multiple criteria, such as ASN, IP addresses, CIDR ranges, countries, and regions (through Labels). For example, you can block traffic from specific countries while creating IP-based exceptions for legitimate business partners within those regions, or combine geographic restrictions with rate limiting for different request volumes per region.</p> 
<p>The following is a sample AWS WAF log showing a geo-blocked request from the United States, including the automatic geographic labeling. If the action is set to COUNT in this rule, then these Labels can be used in downstream Rules as needed.</p> 
<div class="hide-language"> 
 <pre><code class="lang-json">{
    "timestamp": 1755054486205,
    "formatVersion": 1,
    "webaclId": "arn:aws:wafv2:us-east-1:95376154XXX:global/webacl/Test/57d26a7b-66b8-471b-ad07-b32aXXXX",
    "terminatingRuleId": "USBlock",
    "terminatingRuleType": "REGULAR",
    "action": "BLOCK",
    "terminatingRuleMatchDetails": [],
    "httpSourceName": "CF",
    "httpSourceId": "ECB75ZPWSXXX",
    "ruleGroupList": [],
    "rateBasedRuleList": [],
    "nonTerminatingMatchingRules": [],
    "requestHeadersInserted": null,
    "responseCodeSent": null,
    "httpRequest": {
        "clientIp": "3.239.X.X",
        "country": "US",
        "headers": [
            {
                "name": "Host",
                "value": "did07inwxxxx.cloudfront.net"
            },
            {
                "name": "User-Agent",
                "value": "curl/8.11.1"
            },
            {
                "name": "Accept",
                "value": "*/*"
            }
        ],
        "uri": "/",
        "args": "",
        "httpVersion": "HTTP/1.1",
        "httpMethod": "GET",
        "requestId": "VjYFD2a9B2FqXGZ-ljiuLqmDYYBOzNlkglXJ252wJtlyXGXXXlzxxxxx",
        "fragment": "",
        "scheme": "http",
        "host": " did07inwxxxx.cloudfront.net"
    },
    "labels": [
        {
            "name": "awswaf:clientip:geo:country:US"
        },
        {
            "name": "awswaf:clientip:geo:region:US-VA"
        }
    ]
}</code></pre> 
</div> 
<h2>Option 3: AWS Network Firewall</h2> 
<p>AWS Network Firewall operates at the network infrastructure level (layers 3-7), providing geographic IP filtering capabilities that complement application-layer solutions such as CloudFront and AWS WAF. Unlike the previous options that focus on HTTP/HTTPS traffic, Network Firewall can filter all IP, Port, and Protocol, making it ideal for protecting entire VPC workloads regardless of the application protocol.</p> 
<p>When traffic reaches Network Firewall, the service evaluates configured geographic rules and identifies the geographic location of source or destination IP addresses (based on your rule configuration) to apply the appropriate action. These actions include pass, drop, reject, or alert, for traffic to or from specific countries using standard ISO country codes.</p> 
<p>Network Firewall supports geographic filtering through two primary methods:</p> 
<ol> 
 <li>Standard stateful rules: Through the <a href="https://aws.amazon.com/console/">AWS Management Console</a>, you can configure geographic IP filtering with intuitive options to either “<strong>Match only selected countries</strong>” or “<strong>Match all but selected countries</strong>“.</li> 
 <li>Suricata-compatible rules: For advanced users, Network Firewall supports Suricata rule syntax using the “<code>geoip</code>” keyword. This approach enables more sophisticated filtering logic, allowing you to combine geographic restrictions with other packet inspection criteria such as specific protocols, ports, or payload content.</li> 
</ol> 
<p>The following is an example suricata rule set demonstrating how to allow and monitor traffic from a specific country (India). The rule evaluation order was set to Strict.</p> 
<div class="hide-language"> 
 <pre><code class="lang-json">alert ip $EXTERNAL_NET any -&gt; $HOME_NET any (msg:"Ingress traffic from IN allowed"; flow:to_server; geoip:src,IN; metadata:geo IN; sid:202409301;) 
pass ip $EXTERNAL_NET any -&gt; $HOME_NET any (msg:"Ingress traffic from IN allowed"; flow:to_server; geoip:src,IN; metadata:geo IN; sid:202409302;)
</code></pre> 
</div> 
<p>This flexibility enables complex scenarios such as blocking SSH traffic from specific countries while allowing other protocols, or combining geographic restrictions with deep packet inspection rules for enhanced security policies.</p> 
<p>Network Firewall includes the msg keywords in your rules and metadata to provide detailed geographic information in the alert logs. The following is an example alert log generated for the preceding rules:</p> 
<div class="hide-language"> 
 <pre><code class="lang-json">{
    "firewall_name": "Test-NFW",
    "availability_zone": "eu-west-2a",
    "event_timestamp": "1756003042",
    "event": {
        "tx_guessed": true,
        "tx_id": 0,
        "app_proto": "http",
        "src_ip": "43.204.148.X",
        "src_port": 49418,
        "event_type": "alert",
        "alert": {
            "severity": 3,
            "signature_id": 202409301,
            "rev": 0,
            "metadata": {
                "geo": [
                    "IN"
                ]
            },
            "signature": "Ingress traffic from IN allowed",
            "action": "allowed",
            "category": ""
        },
        "flow_id": 799891964035732,
        "dest_ip": "10.0.10.198",
        "proto": "TCP",
        "verdict": {
            "action": "pass"
        },
        "http": {
            "hostname": "18.134.198.X",
            "url": "/",
            "http_user_agent": "curl/8.11.1",
            "http_method": "GET",
            "protocol": "HTTP/1.1",
            "length": 0
        },
        "dest_port": 80,
        "pkt_src": "geneve encapsulation",
        "timestamp": "2025-08-24T02:37:22.969703+0000",
        "direction": "to_server"
    }
}
</code></pre> 
</div> 
<p><strong>Note:</strong> This example and blog focuses on ingress traffic filtering. However, AWS Network Firewall also supports egress geo-blocking, allowing you to control outbound traffic to specific countries. For detailed implementation examples covering both ingress and egress scenarios, along with logging configurations, refer to the&nbsp;<a href="https://aws.amazon.com/blogs/security/aws-network-firewall-geographic-ip-filtering-launch/">AWS Network Firewall Geographic IP Filtering launch blog</a> and the <a href="https://docs.aws.amazon.com/network-firewall/latest/developerguide/rule-groups-geo-ip-filtering.html">Network Firewall documentation</a>.<strong>&nbsp;</strong></p> 
<h2>Option 4: Route 53</h2> 
<p>Route 53 geolocation routing provides DNS-level geographic access control by returning different DNS responses based on the location of DNS queries. This approach enables blocking or redirecting traffic from specific countries before it even reaches your infrastructure, offering a unique upstream filtering capability that operates at the DNS resolution layer.</p> 
<p><a href="https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geo.html">Route 53 geolocation routing</a> functions by mapping IP addresses to geographic locations using geolocation databases. When a DNS query arrives, Route 53 determines the geographic origin based on either the DNS resolver’s IP address or, when available, the client subnet information provided through the <a href="https://www.rfc-editor.org/rfc/rfc7871">EDNS0-Client-Subnet (ECS)</a> extension. The service responds with the appropriate DNS record based on your configured geographic routing rules.</p> 
<p>You can implement Route 53 geolocation blocking through two main strategies:</p> 
<ol> 
 <li>Invalid IP redirection: The most direct blocking approach involves configuring geolocation records for restricted countries to return invalid IP addresses such as 127.0.0.1 (loopback). This causes blocked traffic to fail at the client level without consuming your infrastructure resources, because the requests never leave the user’s system.</li> 
 <li>Error page redirection: A more user-friendly approach directs blocked users to a dedicated error page hosted on CloudFront and <a href="https://aws.amazon.com/s3/">Amazon Simple Storage Service (Amazon S3</a>). This static website can explain the access restriction, provide appeal processes, or offer alternative contact methods. This approach maintains better user experience while clearly communicating access policies.</li> 
</ol> 
<p>Route 53 supports multiple levels of geographic specificity, from continental routing to country-level and US state-level controls. When implementing overlapping geographic regions, the service applies precedence rules where the most specific geographic match takes priority. For example, if you configure both a North America record and a Canada-specific record, then Canadian queries match the Canada record while other North American queries use the continental rule.</p> 
<p>The effectiveness of Route 53 significantly improves when DNS resolvers support <a href="https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-edns0.html">EDNS0-Client-Subnet (ECS)</a>. This extension allows recursive DNS resolvers to include the client’s network subnet information in queries, so that Route 53 can make routing decisions based on the actual client location rather than the DNS resolver location.</p> 
<p>Although many public DNS services such as Google DNS and OpenDNS support EDNS, some do not due to varying privacy and implementation considerations. When EDNS is unavailable, Route 53 falls back to using the DNS resolver’s IP address for geographic determination, which can lead to suboptimal routing decisions when clients use geographically distant DNS servers.</p> 
<h2>Conclusion</h2> 
<p>Each AWS service offers distinct advantages for geo-restriction, with varying levels of complexity, cost, and functionality. The following comparison matrix highlights the key differentiators to help guide your selection process.</p> 
<table border="1" style="border-color: black; border-collapse: collapse;" width="630"> 
 <tbody> 
  <tr style="border: 1px solid black;"> 
   <td style="border: 1px solid black; padding: 8px;">AWS service</td> 
   <td style="border: 1px solid black; padding: 8px;">Service cost</td> 
   <td style="border: 1px solid black; padding: 8px;">Solution setup complexity</td> 
   <td style="border: 1px solid black; padding: 8px;">Inspection on specific URLs</td> 
   <td style="border: 1px solid black; padding: 8px;">Port-Protocol Based Inspection</td> 
   <td style="border: 1px solid black; padding: 8px;">Higher location granularity (for example City, State)</td> 
  </tr> 
  <tr style="border: 1px solid black;"> 
   <td style="border: 1px solid black; padding: 8px;">CloudFront built-in geolocation feature</td> 
   <td style="border: 1px solid black; padding: 8px;">None</td> 
   <td style="border: 1px solid black; padding: 8px;">Low</td> 
   <td style="border: 1px solid black; padding: 8px;">No</td> 
   <td style="border: 1px solid black; padding: 8px;">No</td> 
   <td style="border: 1px solid black; padding: 8px;">No</td> 
  </tr> 
  <tr style="border: 1px solid black;"> 
   <td style="border: 1px solid black; padding: 8px;">CloudFront edge functions</td> 
   <td style="border: 1px solid black; padding: 8px;">Low</td> 
   <td style="border: 1px solid black; padding: 8px;">Medium</td> 
   <td style="border: 1px solid black; padding: 8px;">Yes</td> 
   <td style="border: 1px solid black; padding: 8px;">No</td> 
   <td style="border: 1px solid black; padding: 8px;">Yes</td> 
  </tr> 
  <tr style="border: 1px solid black;"> 
   <td style="border: 1px solid black; padding: 8px;">AWS WAF</td> 
   <td style="border: 1px solid black; padding: 8px;">Low</td> 
   <td style="border: 1px solid black; padding: 8px;">Low</td> 
   <td style="border: 1px solid black; padding: 8px;">Yes</td> 
   <td style="border: 1px solid black; padding: 8px;">No</td> 
   <td style="border: 1px solid black; padding: 8px;">Yes</td> 
  </tr> 
  <tr style="border: 1px solid black;"> 
   <td style="border: 1px solid black; padding: 8px;">Network Firewall</td> 
   <td style="border: 1px solid black; padding: 8px;">Medium</td> 
   <td style="border: 1px solid black; padding: 8px;">Medium</td> 
   <td style="border: 1px solid black; padding: 8px;">Yes</td> 
   <td style="border: 1px solid black; padding: 8px;">Yes</td> 
   <td style="border: 1px solid black; padding: 8px;">No</td> 
  </tr> 
  <tr style="border: 1px solid black;"> 
   <td style="border: 1px solid black; padding: 8px;">Route 53</td> 
   <td style="border: 1px solid black; padding: 8px;">Low</td> 
   <td style="border: 1px solid black; padding: 8px;">Low</td> 
   <td style="border: 1px solid black; padding: 8px;">No</td> 
   <td style="border: 1px solid black; padding: 8px;">No</td> 
   <td style="border: 1px solid black; padding: 8px;">No</td> 
  </tr> 
 </tbody> 
</table> 
<p>For straightforward country-level blocking, start with CloudFront’s built-in geo-restriction or Route 53 geolocation routing—both offer low-cost implementation with minimal configuration. When you need application-aware filtering with URL-specific controls, AWS WAF geo-match rules or CloudFront edge functions provide greater flexibility. For network-level protection that extends beyond web traffic, AWS Network Firewall is the only solution supporting bidirectional inspection across all protocols and ports.</p> 
<p>Most production environments benefit from layering multiple services together. For example, you might use Route 53 to block traffic at the DNS layer, CloudFront’s built-in geo-restriction as a second filter, and AWS WAF for granular, path-specific rules—creating a defense-in-depth architecture that addresses threats at multiple levels.</p> 
<p>By implementing geo-restriction with AWS services, you can:</p> 
<ul> 
 <li>Reduce your attack surface by blocking traffic from high-risk regions before it reaches your applications</li> 
 <li>Lower infrastructure costs by filtering unwanted traffic early, reducing compute, bandwidth, and logging expenses</li> 
 <li>Strengthen your compliance posture by enforcing geographic access policies required by regulatory frameworks such as GDPR or data residency requirements</li> 
 <li>Improve security team efficiency by reducing alert noise and allowing teams to focus on legitimate threats</li> 
 <li>Enhance application performance by reducing the load from unwanted traffic on your origin servers</li> 
</ul> 
<p>Start with the simplest solution that meets your requirements, and add complexity as your geo-restriction needs evolve.</p> 
<p>Get started today by exploring the <a href="https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/georestrictions.html">CloudFront geo-restriction documentation</a>, <a href="https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-geo-match.html">WAF geo-match conditions,</a>&nbsp;<a href="https://docs.aws.amazon.com/network-firewall/latest/developerguide/rule-groups-geo-ip-filtering.html">Network Firewall rules reference</a> and <a href="https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geo.html">Route 53 geolocation routing guide</a>.</p> 
<h2>About the authors</h2> 
<p>
 <!-- First Author --></p> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="NAME OF YOUR IMAGE FROM MEDIA LIBRARY"><img alt="YOUR NAME" class="alignleft wp-image-1288 size-thumbnail" height="HEIGHT IN PIXELS" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/12/19/image_resized.jpg" width="WIDTH IN PIXELS" /></p> 
 <h3 class="lb-h4">Rahi Patel</h3> 
 <p style="color: #879196; font-size: 1rem;">Rahi Patel is a Startups Technical Account Manager at AWS specializing in Networking. He architects cloud networking solutions optimizing performance across global AWS deployments. Previously a Network Engineer with Cisco Meraki, he holds an MS in Engineering from San Jose State University. Outside work, he enjoys tennis and pickleball.</p> 
</div> 
<p>
 <!-- Second Author --></p> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="NAME OF YOUR IMAGE FROM MEDIA LIBRARY"><img alt="YOUR NAME" class="alignleft wp-image-1288 size-thumbnail" height="HEIGHT IN PIXELS" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/12/18/anvkog-profile-picture-125x125-1.jpg" width="WIDTH IN PIXELS" /></p> 
 <h3 class="lb-h4">Anvesh Koganti</h3> 
 <p style="color: #879196; font-size: 1rem;">Anvesh Koganti is a Solutions Architect at AWS specializing in Networking. He focuses on helping customers build networking architectures for highly scalable and resilient AWS environments. Outside of work, Anvesh is passionate about consumer technology and enjoys listening to podcasts on tech and business. When disconnecting from the digital world, Anvesh spends time outdoors hiking and biking.</p> 
</div>
