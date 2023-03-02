
## Setting up AWS Lightsail for incoming WeeWX rsync

AWS Lightsail is one of the cheapest ways to set up a minimal Internet site to rsync to from your LAN WeeWX site.   This page details how I do it, and some pros/cons of alternate mechanisms to get to similar results.

### Background

Many WeeWX users want to be able to access their weather data while away from home.  While there are a variety of ways to do this including punching a hole in your firewall, port forwarding inbound, and setting up a tunnel, all of these expose your home LAN to attack from the thousands of bots and the like out there on Internet.

Having a tiny webserver-only site on Internet that you 'push' data to from your LAN WeeWX site lets you remove all these risks, at a small cost (under $5/month) and perhaps a little bit of work.

The downside is this means your Internet site will also be subject to bot attacks and probes, but hardening your system enough is rather easy to do.  Details are below...


## Setting up an AWS Lightsail VM

### Spin up a VM

* pick the os of your choice and install a minimal os
* save your ssh key to your local client computer so you can ssh in from a terminal windowas that os image's default user (for example 'admin for debian')
* make sure you also install a webserver (I recommend nginx)

Make sure you 'pin' a public ip address to your VM instance so that it has the same ip address after every bootup.  This is set 'off' by default so you have to do this manually in the Lightsail gui.


### Harden your AWS setup

At this point your system should be listening for incoming ssh/22 and http/80 requests from the entire Internet, meaning your VM will almost immediately be found by the thousands of bots running and scanning the public ip address space.  Time to harden your installation....

These steps are done through the lightsail gui.

#### for ssh: permit in only your home public ip address and the AWS console

Edit the default 'SSH TCP 22' rule by clicking the edit icon.  Permit in only the public ip address that is on the Internet-side of your LAN's connection to Internet.  This is likely assigned by your Internet Service Provider. You can find your public ip via doing a google search for 'findmyip'.

Restrict ssh to your home LAN's public ip and (IMPORTANT) be sure that the "Allow Lightsail browser SSH" box is checked, which is your failsafe way to ssh into your VM from other locations after logging into AWS through a browser.

#### for http: consider limiting which countries may access your website

While trying to limit incoming traffic based on the source country that ip address 'seems' to be from, this is a very inexact science since addresses can be faked.  FWIW - I limit my website to only permit in US ip addresses plus a few others.  It seems to help lessen the number of attacks.

Hardening the website is a complicated tasks with many ways of making things better.  Some ways to do so include:
* install fail2ban on the VM and enable some of its filter rules, or even write/enhance your own
* use the webserver's capabilities to do geoip fencing based on country ip ranges
* if you are behind Cloudflare, set up some firewall rules there to turn on anti-bot protections
* or combine the approaches above

You can not 'stop' the bots and their attacks, but by limiting what you run on your Internet VM you can lessen the attack footprint.

Editorial comment - absolutely do NOT run WordPress nor ideally anything requiring PHP.  The number of attacks looking for holes in such frequently-misconfigured software is a bit amazing.  I saw 'thousands' of probes in a week for such things before I locked a test VM down much tighter.

#### Enable automatic os updates

If you run Ubuntu you can easily set it to download and even install updates automatically, although you will have to periodically log in via ssh and see if it tells you 'reboot required'.   See the available Ubuntu resources for how to do this.

Other os can be set similarly.  Consult your os's documentation for what it can/can't do and how to set that up.


----

At this point you have a running VM 

### Set up WeeWX rsync


Follow the normal [StdReport] [[RSYNC]] setup steps
* set up a ssh keypair to use on the WeeWX client computer
* add that keypair's *public* key to `.ssh/authorized_keys` in the Lightsail VM account 
* set up ~/.ssh/config in the WeeWX system's account
* edit the RSYNC stanza in weewx.conf to match
* restart WeeWX

After the archive_period (typically 5 minutes) your logs should show a successful rsync.

### Put your website behind https

There are multiple ways to do this, each with pros/cons.

1. Put your site behind Cloudflare
2. Install your own certificates with LetsEncrypt


----

## Further hardening options

### Go to https for your website

#### Cloudflare

The easiest way for this is to set up a free Cloudflare account, and put your site under Cloudflare for DNS.   The benefit of this is that it is very easy to set up.  The downside is that you will see all traffic in your webserver logs as coming 'from' Cloudflare (which is is), meaning you cannot block anything on the VM at the os level with things like fail2ban. 

If you go Cloudflare, you should use the Lightsail gui to limit http/80 into your VM to come from 'only' Cloudflare.   Add rules for http/80 that permit in only the published Cloudflare addresses.  At this writing the ranges are published at https://www.cloudflare.com/ips/ - you might also let your home ip to be permitted in (for debugging purposes)

Cloudflare limits you to five (5) firewall rules so you can only do limited blocking of things.

FWIW my test system has the following rules:
* block anything with 'wp' in the URL (block WordPress related probes)
* permit in only US and a few other countries
* block everything else

#### Lets Encrypt

A slightly more complicated way is to enable https by getting a cert from https://letsencrypt.org and setting your nginx to be https only.   Benefit here is that you 'will' see the source ip addresses and you 'can' use fail2ban to block them automatically if they match any enabled filter rules.  Downside is that all the attempts will make it to your VM for blocking there, but I have not seen this as a problem in several years of running this way.






