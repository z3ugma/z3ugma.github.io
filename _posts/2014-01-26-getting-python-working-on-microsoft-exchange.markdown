---
title: Getting Python Working on Microsoft Exchange
date: 2014-01-26 22:33:56 Z
layout: post
comments: true
---

### Creating an Internal Emailing Wikibot in Python ###

You know how they say that when all you have is a hammer that everything looks like a nail?

For now, Python is my hammer. I understand that C# or VB would be a better choice for this project, particularly since Microsoft Exchange plays much more nicely with them than Python, but as I'm just starting out I think I'm still in the phase of learning  _how the language works_, and haven't quite progressed to learning _what the language is good for_. 

I suppose there's something to be said for the fact that Python is platform-agnostic.

Here's a project I'm working on, though: the first end-goal is to:

1.)   Find a random wiki page
2.)   Find its owner
3.)   Email that owner to notify them that page was chosen

### Part 1: Connecting Python to Exchange Web Services ###

Since this is an internal wiki to my company, there is a lot of benefit to using Exchange to send the emails. Most of all that I am rolling it on my own, and not working with the network admins and therefore don't have access to the SMTP server.

Microsoft Exchange Server has a SOAP API, called Exchange Web Services. This has been a fun experience learning SOAP for the first time.

#### What is SOAP? ####
SOAP stands for Simple Object Access protocol. It's an XML-based messaging system that servers and clients can use to talk to each other with standardized messages.  The client understand what XML tags the server uses by getting a file called the WSDL.

I'll be using a Python SOAP library called Suds  to talk to the Exchange server.

There are a few catches, though. EWS doesn’t play nice with Suds, so we need a few patches and other modules to get it working well. 

### Installing Suds ###

On Mac and Linux, this is trivially easy: run 
{% highlight bash %}
sudo pip install suds
{% endhighlight %}

(If you don’t have pip on your Mac, you can run [the get-pip](https://raw.github.com/pypa/pip/master/contrib/get-pip.py) installer file.

On Windows, where I did it, this is a little harder, but you can still download an easy_install script and the zip file of Suds.

Then, you boot up a SOAP client with Suds, and I didn't document this very well, but it returns an HTTP 401 error; Not Authorized.

### Getting Suds to Work with EWS ###

Hmm. Googling for a few hours brought me to a [good solution, first suggested on the Suds forums](http://mail.libexpat.org/pipermail/soap/2011-September/000583.html). 

EWSClient is Daniel Holth's solution to marrying Suds to EWS.  Installing it will also intall python-ntlm, which will be important later.

The main file in EWSClient is:

{% highlight python %}
import suds.client
import suds.plugin
import suds.store
import urlparse

class EWSClient(suds.client.Client):
    pass

class AddService(suds.plugin.DocumentPlugin):
    # WARNING: suds hides exceptions in plugins
    def loaded(self, ctx):
        """Add missing service."""
        urlprefix = urlparse.urlparse(ctx.url)
        service_url = urlparse.urlunparse(urlprefix[:2] + ('/EWS/Exchange.asmx', '', '', ''))
        servicexml = u'''  <wsdl:service name="ExchangeServices">
    <wsdl:port name="ExchangeServicePort" binding="tns:ExchangeServiceBinding">
      <soap:address location="%s"/>
    </wsdl:port>
  </wsdl:service>
</wsdl:definitions>''' % service_url
        ctx.document = ctx.document.replace('</wsdl:definitions>', servicexml.encode('utf-8'))
        return ctx
{% endhighlight %}

This adds a plugin to Suds that adds EWS's definitions to the end of the SOAP request. I don't quite understand how it works, given Microsoft's byzantine EWS definitions. At any rate, it works.

EWSClient has a few other files, too, like monkey.py, which fixes an issue where the client makes a request to the often-overloaded W3C definitions for XML. It instead lets you access a locally-cached copy of the file. I disabled it because my Windows python client was causing me a headache.

Using the example files that ewsclient provides, I try to get the WSDL specs:

{% highlight python %}
import ewsclient
import os
import suds.client
from suds.transport.https import WindowsHttpAuthenticated
import logging

#logging.basicConfig(level=logging.DEBUG)

def test_basic():
    domain = 'exhange_server_url_goes_here'
    username = r'DOMAIN/username'
    password = 'password'

    transport = WindowsHttpAuthenticated(username=username,
            password=password)
    client = suds.client.Client("https://%s/EWS/Services.wsdl" % domain,
            transport=transport,
            plugins=[ewsclient.AddService()])

    return client

print test_basic()

{% endhighlight %}

And get this error as a response:

{% highlight bash%} 
Traceback (most recent call last):
  File "test_auth.py", line 18, in <module>
    response = urllib2.urlopen(url)
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/urllib2.py", line 127, in urlopen
    return _opener.open(url, data, timeout)
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/urllib2.py", line 410, in open
    response = meth(req, response)
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/urllib2.py", line 523, in http_response
    'http', request, response, code, msg, hdrs)
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/urllib2.py", line 442, in error
    result = self._call_chain(*args)
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/urllib2.py", line 382, in _call_chain
    result = func(*args)
  File "/Library/Python/2.7/site-packages/ntlm/HTTPNtlmAuthHandler.py", line 99, in http_error_401
    return self.http_error_authentication_required('www-authenticate', req, fp, headers)
  File "/Library/Python/2.7/site-packages/ntlm/HTTPNtlmAuthHandler.py", line 35, in http_error_authentication_required
    return self.retry_using_http_NTLM_auth(req, auth_header_field, None, headers)
  File "/Library/Python/2.7/site-packages/ntlm/HTTPNtlmAuthHandler.py", line 69, in retry_using_http_NTLM_auth
    (ServerChallenge, NegotiateFlags) = ntlm.parse_NTLM_CHALLENGE_MESSAGE(auth_header_value[5:])
  File "/Library/Python/2.7/site-packages/ntlm/ntlm.py", line 217, in parse_NTLM_CHALLENGE_MESSAGE
    msg2 = base64.decodestring(msg2)
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/base64.py", line 321, in decodestring
    return binascii.a2b_base64(s)
binascii.Error: Incorrect padding

{% endhighlight %}

The thing to focus on is that we're authenticating with NTLM, and the auth_header_value s aren't formatted properly. There are some issues in python-ntlm that need fixing to play nicely with the encoding of domain, user, and pass.

### Fixing NTLM to Work with our Suds Client ###

[There's an issue documented at the python-ntm page](https://code.google.com/p/python-ntlm/issues/detail?id=17). The solution in post #3, in which we add a few lines to the code of /Library/Python/2.7/site-packages/ntlm/HTTPNtlmAuthHandler.py (on my Mac, your filepath will be different), worked for me:

{% highlight bash%}
--- HTTPNtlmAuthHandler_old.py  2012-03-19 15:29:08.503699995 +0100
+++ HTTPNtlmAuthHandler.py      2012-03-19 15:30:22.459242446 +0100
@@ -66,6 +66,8 @@
                 headers['Cookie'] = r.getheader('set-cookie')
             r.fp = None # remove the reference to the socket, so that it can not be closed by the response object (we want to keep the socket open)
             auth_header_value = r.getheader(auth_header_field, None)
+            if ',' in auth_header_value:
+                auth_header_value, postfix = auth_header_value.split(',', 1)
             (ServerChallenge, NegotiateFlags) = ntlm.parse_NTLM_CHALLENGE_MESSAGE(auth_header_value[5:])
             user_parts = user.split('\\', 1)
             DomainName = user_parts[0].upper()

{% endhighlight %}

With these changes, made, testing authorization works and I'm ready to start doing things with my client.
