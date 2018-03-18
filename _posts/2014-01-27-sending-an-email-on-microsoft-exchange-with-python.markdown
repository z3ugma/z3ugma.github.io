---
title: Sending an Email on Microsoft Exchange with Python
date: 2014-01-27 00:20:09 Z
layout: post
comments: true
---

### Part 2: Getting the Python EWS Client to Send an Email ###

Now that I've got a client connect to the Exchange server, I can actually use the SOAP API methods as documented in the WSDL and on [Microsoft's documentation](http://msdn.microsoft.com/en-us/library/aa580675.aspx).

Suds has great built-in methods and classes for working with SOAP, but [as this post confirms](https://lists.fedoraproject.org/pipermail/suds/2010-July/001029.html), bugs in both Suds and EWS mean that I'll have to manually build the XML and inject it directly into the message.

So far, I have code that will  connect a Suds client to my exchange server and send an XML message:

{% highlight python %}

import ewsclient #Changes to Suds to make it EWS compliant
import ewsclient.monkey # More Suds changes
import datetime
import os
import sys
import suds.client
import logging
import time
from suds.transport.https import WindowsHttpAuthenticated

#Uncomment below to turn on logging
#logging.basicConfig(level=logging.DEBUG)

#Logging on/accessing EWS SOAP API
domain = 'exchange server'
username = r'DOMAIN\username'
password = 'password'

transport = WindowsHttpAuthenticated(username=username,
        password=password)
client = suds.client.Client("https://%s/EWS/Services.wsdl" % domain,
        transport=transport,
        plugins=[ewsclient.AddService()])

#Now that the SOAP client is connected to EWS, send this XML soap message
client.service.CreateItem(__inject={'msg':xml})

{% endhighlight %}

Now, we just need to tell it what to send in that message.

### Building the XML Message ###

There are all kinds of things you can talk to EWS about, but I just want to talk about emails.

I started with a [template message from Microsoft](http://msdn.microsoft.com/en-us/library/office/aa566468.aspx) and changed a few things.

First,  I wanted to place a copy in my SentItems folder, so I added

{% highlight xml %}
 <SavedItemFolderId>
        <t:DistinguishedFolderId Id="sentitems" />
 </SavedItemFolderId>
{% endhighlight %}

In my first iteration, I had the body sent as text but I wanted to replace it with an HTML email template to make it prettier. This was tough for me to figure out; I kept getting XML validation errors. Eventually, I learned about [CDATA](http://www.w3schools.com/xml/xml_cdata.asp), a tag that tells the XML parser to ignore whatever's inside it.

I replaced the Body tag with:

{% highlight xml%}
<t:Body BodyType="HTML"><![CDATA[#Body#]]></t:Body>
{% endhighlight %}

And will replace the #Body# with some HTML from an email template later.
Lastly, I put in some information about an automatic Reminder to get the final message:
{% highlight xml%}
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"
  xmlns:t="http://schemas.microsoft.com/exchange/services/2006/types">
  <soap:Body>
    <CreateItem MessageDisposition="SendAndSaveCopy" xmlns="http://schemas.microsoft.com/exchange/services/2006/messages">
      <SavedItemFolderId>
        <t:DistinguishedFolderId Id="sentitems" />
      </SavedItemFolderId>
      <Items>
        <t:Message>
          <t:ItemClass>IPM.Note</t:ItemClass>
          <t:Subject>Python EWS Bot #SubjectDate#</t:Subject>
          <t:Body BodyType="HTML"><![CDATA[#Body#]]></t:Body>
		  <t:ReminderDueBy>#ReminderDate#T11:00:00-06:00</t:ReminderDueBy>
          <t:ReminderIsSet>1</t:ReminderIsSet>
          <t:ReminderMinutesBeforeStart>180</t:ReminderMinutesBeforeStart>
          <t:ToRecipients>
            <t:Mailbox>
              <t:EmailAddress>email@domain.com</t:EmailAddress>
            </t:Mailbox>
          </t:ToRecipients>
          <t:IsRead>false</t:IsRead>
        </t:Message>
      </Items>
    </CreateItem>
  </soap:Body>
</soap:Envelope>
{% endhighlight %}

Each of those #Variable# tags I put in the XML will be string-replaced on the Python side.

### Putting Variables into the Message ###

I wrote a quick little function to take a list of tuples and a template text and replace the text in the template:
{% highlight python %}
#Replacing the text in the XML SOAP message
def xmlreplace(text,list):
    for i in list:
        if i[0] in text:
            text = text.replace(i[0],str(i[1]))
    return text

#Make a string out of the current time
curdatetime =  time.strftime("%c")

#Set the Reminder Date for one week from today
reminderday = datetime.date.today()+datetime.timedelta(days=7)

#Open an HTML file to use as the body
template = (open(file, 'r'))
    body = template.read()

#List of parameters in the XML to replace: (template, replacement).
xmlparams=[('#SubjectDate#',curdatetime),('#Body#', body),('#ReminderDate#',reminderday)]

# Replace everything in the XML template shown above with the new dynamic values
xml = xmlreplace(xmltemplate,xmlparams)
{% endhighlight %}

Now that I have XML as a big string, I can send my message:

<img src="{{ root_url }}/images/pythonewsemail.png" />