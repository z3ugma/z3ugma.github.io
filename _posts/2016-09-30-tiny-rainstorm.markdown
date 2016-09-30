---
layout: post
title: "Tiny Rainstorm"
date: 2016-09-30 12:35:00 -0500
comments: true
published: true
---

I like listening to [RainyMood](http://www.rainymood.com) while I work to keep focused. The site isn't that big, just 327kb, but its usability isn't great and it's full of tracking code. The site hits my adblocker(s) (plural at work per GPO) and so is slow to load.

With some "inspect element" I found the source files and made a tiny (just 35k) version that loops, is visually attractive, doesn't track me, and loads instantly.

You can view the published webpage [here](/files/tinyrain.html) or [download the raw file to use on your site](https://raw.githubusercontent.com/z3ugma/z3ugma.github.io/master/files/tinyrain.html)

{% highlight html %}
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html xml:lang="en" xmlns="http://www.w3.org/1999/xhtml" lang="en">
    <head>
        <meta http-equiv="content-type" content="text/html; charset=UTF-8" />
        <meta name="robots" content="noindex, nofollow" />
        <meta name="googlebot" content="noindex, nofollow" />
        <script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/mootools/1.6.0/mootools-core-compat.min.js"></script>
        <title>TinyRain</title>

        <script type='text/javascript'>
            window.addEvent('load', function() {
            var song = new Audio("http://rainymood.com/audio1110/0.m4a");
            song.controls = true;
            song.loop = true;
            song.autoplay = true;
            document.body.appendChild(song);
            });
        </script>
        
        <style>
            html, body {
                height: 100%;
            }

            html {
                display: table;
                margin: auto;
            }

            body {
                display: table-cell;
                vertical-align: middle;
            }
        </style>

        <link rel="shortcut icon" href="https://cdn3.iconfinder.com/data/icons/weather-16/256/Rainy_Day-16.png">
    </head>
    <body>
    </body>
</html>
{% endhighlight %}
