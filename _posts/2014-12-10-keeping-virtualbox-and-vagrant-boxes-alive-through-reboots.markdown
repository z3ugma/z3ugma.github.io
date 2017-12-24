---
title: Keeping VirtualBox and Vagrant Boxes Alive Through Reboots
date: 2014-12-10 18:50:55 Z
layout: post
comments: true
---

I recently switched from a dedicated Windows 7 PC for my home server to a Mac Mini, mostly for the better electricity consumption and the fact the the PC was having nightly bluescreen crashes and restarting.

I have always been a fan of RDP, and my office uses PCs - so to keep the convenient RDP access to home, I installed [VirtualBox](www.virtualbox.org) and created a Windows VM. This has a bridged network adapter, so it just looks like another computer on my home network.

However, when Mac OS restarts, or after a power failure, the virtual machine is powered off. This won't do.

##Daemons in Mac OS##

Mac OS has the usual suspects like cron, but has a neat daemon launching system, appropriately called [launchd](https://developer.apple.com/library/mac/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html), introduced in 10.4.

Launchd works by "loading" (think of it like a soft-install) objects called '''plist'''. Plist is a serialized object format like JSON or XML that tells launchd the properties of how to execute a particular daemon.

If you want to play with creating your own plists, head over to [http://launched.zerowidth.com](http://launched.zerowidth.com), where Nathan Witmer has created a plist generator.

##Automatic Tasks in VirtualBox##

VirtualBox comes with a command-line interface to automate tasks on VMs. My need is simple - just boot the box:

{% highlight bash %}
VBoxHeadless -s Windows -v on
{% endhighlight %}

This follows the syntax for VBoxHeadless:

{% highlight text %}
  -s, -startvm, --startvm <name|uuid>   Start given VM (required argument)
  -v, -vrde, --vrde on|off|config       Enable (default) or disable the VRDE server or don't change the setting
{% endhighlight %}

VRDE is the Virtual Remote Desktop extension, which allows RDP out of the box through a special Oracle tool.

##Booting my VM at Login##

launchd has multiple "runlevels" - there are System level daemons, and daemons for whenever a given user logs in. User daemons are stored at ~/Library/LaunchAgents/.

With the help of the launched tool, I made a plist for my command:


{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>KeepAlive</key>
    	<true/>
	<key>Label</key>
	<string>io.zzzz.z3ugma.launched.windowsvirtualbox</string>
	<key>ProgramArguments</key>
	<array>
		<string>VBoxHeadless</string>
		<string>-s</string>
		<string>Windows</string>
		<string>-v</string>
		<string>on</string>
	</array>
	<key>RunAtLoad</key>
	<true/>
	<key>UserName</key>
	<string>fred</string>
	<key>WorkingDirectory</key>
	<string>/Users/fred</string>
	<key>StandardErrorPath</key>
    	<string>/Users/fred/windowsvm.log</string>
    	<key>StandardOutPath</key>
    	<string>/Users/fred/windowsvm.log</string>
</dict>
</plist>
{% endhighlight %}

Notable options in here:

* **KeepAlive = True** - If the VBoxHeadless process crashes for some reason, or someone manually shuts down the box (perhap from within RDP), the box restarts itself
* **RunAtLoad = True** - Run this daemon when the daemon is loaded. "Loading" occurs at login, and when a user manually loads the daemon.
* **WorkingDirectory** & **UserName** - These are special directives needed because of the peculiar way that VirtualBox runs. Just set it to the home folder of the user running the daemon.

This xml is saved into a file in /Library/LaunchAgents. Navigate to that directory, and execute

{% highlight text %}
launchctl load <name of plist>
{% endhighlight %}

*launchctl* is the program that Mac OS uses to control launchd processes. Once the plist has been loaded, it should persist after reboot.

A similar plist can be used for the command 'vagrant up' to launch vagrant vms.

