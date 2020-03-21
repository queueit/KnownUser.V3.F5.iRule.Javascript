# F5 JAVASCRIPT INJECTION
When properly configured, *F5 iRules* utilize a scripting syntax which allows the load balancer to intercept, inspect, transform, and direct inbound or outbound application traffic.
In the event the application's source is unable to be modified and all other forms of injection have failed or are not applicable, clients with a F5 load balancer can request their network security team configure an *iRule* to intercept an application's response and inject HTML into the page source necessary for enabling Queue-it. This is similar to the format that manual injection uses, but HTML is inserted into the webserver's response rather than into the original source code.
For instructions on how to build, deploy, and test an *iRule*, we recommend you consult your F5 support team and network security team as they are most knowledgeable and equipped to handle such a request.

## Generic F5 iRule Template (**requires further customization**)
```
when HTTP_REQUEST 
  { 
   # This is the condition for which requests will be matched against 
   if {[HTTP::uri] contains "segment/in/uri"} { 
    set enableEum 1 
    } 
    else { 
    set enableEum 0 
    } 
   # Disable the stream filter for client requests as we are only interested in the server response 
   STREAM::disable 
   # LTM does not uncompress response content, so if the server has compression enabled 
   # and it cannot be disabled on the server, we can prevent the server from 
   # sending a compressed response by removing the compression offerings from the client
   # HTTP::header remove "Accept-Encoding"
  } 
    
 when HTTP_RESPONSE 
  { 
   # Disable the stream filter for all server responses 
   STREAM::disable 
   # Inserts the necessary JavaScript for EUM
   if {($enableEum == 1) && ([HTTP::header "Content-Type"] starts_with "text/html")} {
    STREAM::expression {@</title>@</title> 
    <script type="text/javascript" src="//static.queue-it.net/script/queueclient.min.js"></script>
    <script 
      data-queueit-c="[YOUR-CUSTOMER-ID]" 
      type="text/javascript" 
      src="//static.queue-it.net/script/queueconfigloader.min.js">
     </script>@} 
    # Enable the stream filter for this response only 
    STREAM::enable
   }
  }
```
## How to use this template
### Matching Condition - The matching condition in your first "if statement."
Change the condition in the above template to match inbound traffic to your application. In the template we match against segments in the application's URI. However, there are other properties which can be used (i.e `[HTTP::path]`, `[HTTP::host]`, etc.): (requires further customization)
>`[HTTP::uri] contains "segment/in/uri" `

### Compression - Does your application use compression?
If you are unsure if compression is being used, there is a simple Curl test to find out. The results will return one of three responses: 1) "Content-Encoding: gzip". 
2) "Content-Encoding: deflate", or 
3) nothing. 
If the first or second response comes back that means the site is using compression, while an empty response means that compression is not being used.

Test: (**requires further customization**)
>`curl -sILH 'Accept-Encoding: gzip,deflate' www.yoursystem.com | grep 'Content-Encoding'`

Response:
>`Content-Encoding: gzip`

If compression is being used, uncomment the following line from the template.
` HTTP::header remove "Accept-Encoding"`

If there isn't any compression, then keep the line commented out or remove it from the rule completely. 
`# HTTP::header remove "Accept-Encoding"`
 
### JavaScript option – What Queue-it account / CustomerId?
You need to insert your Queue-it account / CustomerID in the rule. You can see the CustomerID in you GO Queue-it account https://go.queue-it.net/companyprofile. Do also visit Queue-it’s Technical Integration White paper for information on how to connect the sript to the Integration | Queue-it.
JavaScropt option: (**requires further customization**)
```javascript
<script type="text/javascript" src="//static.queue-it.net/script/queueclient.min.js"></script>
<script 
  data-queueit-c="[YOUR-CUSTOMER-ID]" 
  type="text/javascript" 
  src="//static.queue-it.net/script/queueconfigloader.min.js">
</script>
```

### Other Points To Consider 
* In some cases, the Stream profile must re-enable response re-chunking as documented in https://support.f5.com/csp/article/K6422
* Although the above script template is typically successful, every client environment is different. Therefore, our team recommends that you consult a trained engineer responsible for managing your F5 load balancer. We also highly recommend testing in a sandbox or development environment before deploying any changes to your production environment

## Results
If properly configured, the *iRule* matches a condition for your application-specific traffic, the load balancer will inject the Queue-it JavaScript into the response received by the browser allowing the JavaScript snippet and the associated configuration to load in the browser, read queue status, and send the end-user into queue if needed.
