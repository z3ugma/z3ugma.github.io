---
title: Using cURL To Test Exchange Web Services
date: 2014-01-26 23:02:32 Z
layout: post
comments: true
---

From [this post](http://blogs.msdn.com/b/exchangedev/archive/2009/02/05/quick-and-dirty-unix-shell-scripting-with-ews.aspx), I learned how to write a little shell script that uses cURL to check to see if I'm connecting to the Exchange Web Services server correctly. This made testing my Python EWS script easier, especially when I accidentally connected to the wrong WiFi network and couldn't contact the EWS server.

{% highlight bash %}
curl -u username@domain:password -L https://exchange_server/ews/exchange.asmx -H "Content-Type:text/xml"    --ntlm
{% endhighlight %}

This will read back the schema from the WSDL file if you've successfully connected to the server.
