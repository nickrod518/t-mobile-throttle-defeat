# Stop T-Mobile throttling!
Let me first state I do not know if this goes against T-Mobile's Terms of Service and I do not condone doing so - this is simply a learning project I used to better understand T-Mobile's data throttling methods.

Here's how I used a raspberry Pi to overcome T-Mobile throttling on the One plan. This will enable you to use your home router with many devices connected behind it to access the internet through your T-Mobile sim card, without experiencing throttling.

## How T-Mobile throttles currently
T-Mobile uses the time-to-live value of packets to determine if they have been routed through a phone or originate from the phone itself. To circumvent this, you want your tethered traffic to have the same TTL as phone traffic. The idea is to tether a device capable of overwriting TTL and set it to +1 over what you expect the phone's TTL to be, so that when it is routed by the phone and the TTL is decremented by 1 it is then the expected value.

Most phones have a TTL of 64. This means we need our tethered device's TTL to be 65, so that when it is decremented by passing through the phone it has the identical value of 64 and cannot be differentiated.

# Raspberry Pi to the rescue
We'll be configuring the Pi to act as a bridge and then using IP Tables to overwrite the TTL on each packet. For our purposes eth0 is the adapter we're connecting to the modem and eth1 will be connected to our router's WAN port.

This tutorial assumes you've installed Raspbian Stretch Lite and you're logged into the console (e.g. not ssh'd)

    sudo apt update && sudo apt upgrade -y && sudo apt install rpi-update dnsmasq iptables-persistent -y && sudo rpi-update

## Configure a static IP

I suppose you could just turn on the wifi hotspot on your android phone.  I found it more reliable to pick up a hotspot device. I'm using a Netgear LB1121 I got on Amazon for about $120 along with a generic USB-to-Ethernet adapter.

Give the ethernet connection a static ip:

    sudo nano /etc/dhcpcd.conf

and add the following to the top of the file:

    interface eth1
    static ip_address=192.168.6.1/24

## Setting up dnsmasq

dnsmaq will bridge our wifi and ethernet connections.  
let's backup what's there:

    sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
    sudo nano /etc/dnsmasq.conf
    
Now add the following:

    interface=eth1 # Use interface eth0 
    listen-address=192.168.6.1 # Explicitly specify the address to listen on 
    bind-interfaces # Bind to the interface to make sure we aren't sending things elsewhere 
    server=1.1.1.1 # Forward DNS requests to Cloudflare's DNS
    domain-needed # Don't forward short names 
    bogus-priv # Never forward addresses in the non-routed address spaces. 
    dhcp-range=192.168.6.50,192.168.6.150,12h # Assign IP addresses between 192.168.6.50 and 192.168.6.150 with a 12 hour lease time

## enable IPv4 forwarding

    sudo nano /etc/sysctl.conf

Uncomment the following line:

    net.ipv4.ip_forward=1

## IP Tables
(This is where the magic happens)

    sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    sudo iptables -t mangle -A POSTROUTING -j TTL --ttl-set 65

We need these to persist after reboot so let's use the package we installed earlier to save them:

    sudo netfilter-persistent save

Go ahead and reboot: 

    sudo reboot
    
All you devices should be using your T-Mobile connection without being throttled.
