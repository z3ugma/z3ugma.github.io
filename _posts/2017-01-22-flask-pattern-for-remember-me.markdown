---
layout: post
title: "A Flask Application Pattern for Remember Me"
date: 2018-03-17 17:35:00 -0500
comments: true
published: false
imageurl: ""
description: "'Remember Me' is hard to get right, and Flask-Login is too cumbersome for me needs. Here's my pattern for Remember Me functionality in my Flask apps"
---

It's irritating to remember passwords for websites you visit infrequently. Those passwords are not accessible right at the front of your mind, because you rarely need them.

It's doubly irritating to forget the password to a site you view as "set it and forget it" - the kind of website where you'll only need to log in once a year, but when you need to log in __you really need to log in__. 

Like MyChart for a young, healthy patient, or your cell phone provider when you autopay the bill.

"Remember Me" is an essential piece of functionality for these kinds of websites. I'll show you how I set this up in Flask.

### "Remember Me" is Hard

According to the internet, it's difficult to make a nice, solid Remember Me functionality. The really tough part for me was implementing a corresponding "Forget Me" functionality - you want the user to be able to force a logout of other sessions when they feel their security has been compromised.

You want to give the client enough information to remember the user, but not so much that the user's personal details are divulged. And you want to make sure the user can be remembered on only certain devices and not others, like public library computers.

In order to implement "Remember Me", you need the client to have a place to store something - the "memory" that gets "remembered", if you will.

In the browser, the best place to do this is the ___cookie___.

### Setup

