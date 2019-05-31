---
layout: post
category : lessons
tagline: "Intercepting network traffic on iOS"
description: mitm with mitmproxy on iOS - Kolin Stürt.
tags : [iOS, mitmproxy, mitm, macOS, pf, security, XCode]
---
{% include JB/setup %}

## Intercepting Network Traffic on iOS

We are going to use [mitmproxy](https://mitmproxy.org/) to intercept HTTPS traffic on an iOS device. HTTPS normally validates the certificates in the chain against the ones that are already trusted by iOS. We are going to instruct iOS to accept a custom mitmproxy certificate. This is similar to the concept of when a device is provisioned with a corporation's certificate. It allows the corporation to intercept the device's communications. OpenBSD includes a tool called pf packet filter that is used for NAT and filtering of TCP/IP traffic. Since Darwin is derived from BSD, OS X has the pf packet filter too, at least since Lion. mitmproxy uses the tool to implement it's transparent mode. In order to use mitproxy, you will need to install the mitmproxy certificates on the test device.

### Installing

Let's start by configuring OS X to work with mitmproxy. In the terminal, copy/paste the following to turn on IP forwarding.

    sudo sysctl -w net.inet.ip.forwarding=1
  
 Next, add the following lines into a file called pf.conf (vi pf.conf, for example)
 
    rdr on en2 inet proto tcp to any port 80 -> 127.0.0.1 port 8080
    rdr on en2 inet proto tcp to any port 443 -> 127.0.0.1 port 8080
  
  What these two lines do is inform pf to redirect the traffic for port 443 and 80 to your local on port 8080. Our mitmproxy instance will be running on port 8080. Note that en2 may differ for your configuration. It should be the the interface where the test device will connect on.
  
  Now tell pf to use this configuration.
  
    sudo pfctl -f pf.conf
    
To turn on pf, use this command:

    sudo pfctl -e
 
 pfctl allows inspection of the state table. In order for mitmproxy to be able to use pfctl, we need to edit /etc/sudoers on OS X as root.  At the end of the file, put this:
 
    ALL ALL=NOPASSWD: /sbin/pfctl -s state
  
 This allows any user on OS X to execute /sbin/pfctl -s state as root without a password so you should remove it when you are finished testing.
 
 ### Running
 
 You can start mitmproxy like this:
 
    ./mitmproxy --host
  
 -T flag is for transparent mode.
--host means to use the the Host header for the URL display.

To configure the test device to route traffic through the mitmproxy instance, first get the ip of your OS X host.

    ifconfig
  
On iOS, go to Settings -> Wifi -> HTTP Proxy -> Manual. Enter the ip you got from ifconfig and set the port to 8080.
 
Now the traffic on your device should start showing up in the mitmproxy application.
 
To make sure no one else takes advantage of mitmproxy, uninstall the mitmproxy certificates from the device when you are finished testing.

Settings -> General -> Profiles -> Delete the profiles
