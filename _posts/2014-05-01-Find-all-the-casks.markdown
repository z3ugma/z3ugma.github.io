---
layout: post
title: "Find all the casks"
date: 2014-05-01 12:50:55 -0300
comments: true
categories: 
---

There's a great package manager for Mac called Homebrew that acts sort of like Aptitude on Ubuntu; it makes it really easy to quickly install software for your Mac from the command line.

The Casks platform is built on top of Homebrew - it allows you to install your commonly-used desktop apps for Mac. This makes writing a setup script for when you reformat easy.

It also makes Casks a great place to explore to find new Mac software you didn't know existed. Unfortunately, the most you can see from the Casks master list on Github is the name of the software - you have to go into each .rb to find the URL for the website of the software.

I wrote a quick and dirty script to go through all of these Casks and make a text file that I can put in Excel to open all the websites at once.

First, we download all the files in the github repo to our target folder, then we run:

{%highlight python %}

from os import listdir
from os.path import isfile, join

mypath = '/path/to/target/folder'
onlyfiles = [ f for f in listdir(mypath) if isfile(join(mypath,f)) ]

d = {}
i = 0

for file in onlyfiles:
    with open(file, 'r') as f:
        d[i]=(f.read()).split()
        f.close()
        i = i + 1

with open('casks.txt', 'wb') as dfile:
    for t in d.values():
        text = t[1] + '\t' + t[5] + '\t' + t[7] + '\n'
        text = text.replace('\'','')
        print text
        dfile.write(text)
    dfile.close()

{%endhighlight%}

Now we have casks.txt, ready to import into Excel to easily open all the links within and browse for new Mac apps.