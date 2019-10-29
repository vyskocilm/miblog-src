---
title: "qrget: easy to use privacy oriented cloudless file sharing"
date: 2019-10-24T22:25:27+02:00
draft: true
github: https://github.com/vyskocilm/qrget
---

## The Problem

How to transfer files from my laptop to my cell phone.  
[![qrget screenshot](https://raw.githubusercontent.com/vyskocilm/qrget/master/doc/screenshot.png)](https://github.com/vyskocilm/qrget#installation)

## Why is this a problem?

Actually there are plenty of solutions there

*   Cloud based sharing: Google Disc, Dropbox, (Next|Own)Cloud, old fashioned FTP, SFTP, sending emails, Facebook Messenger, ...
*   Bluetooth
*   NFC
*   USB transfer
*   ...maybe there are more options there ...

The key factor is that except Cloud based methods I would not call others as simple to use. I saw too many times Bluetooth unable to connect between various devices, or there are some machines, which lacks Bluetooth or NFC completely. USB cable is easier. It's almost always plugged in order to charge the phone, isn't it? But hell, what is [MTP, PTP and where is my loved USB Mass Storage mode](https://www.howtogeek.com/192732/android-usb-connections-explained-mtp-ptp-and-usb-mass-storage/)?  
For a Cloud ... it simply does not feel right that we can't use decentralized technology (TCP/IP, HTTP) without a dependency on centralized server(s) somewhere. Geek inside me shouts: **Devices MUST talk directly**.

## The past

I simply run Python builtin http server to server the files. I found the process as easy until I saw the empty faces of others seeing the steps I was doing:

1.  Run [Python](https://python.org) builtin http server somewhere

    <pre>python3 -m http.server
    </pre>

2.  Find IP address using `ip adds show` in a similarly looking blob of text

    <pre>2: wlp2s0: <broadcast,multicast,up,lower_up>mtu 1500 qdisc mq state UP group default qlen 1000
        link/ether 10:f0:05:07:75:ab brd ff:ff:ff:ff:ff:ff
        inet 192.168.1.203/24 brd 192.168.1.255 scope global dynamic noprefixroute wlp2s0
           valid_lft 40987sec preferred_lft 40987sec
        inet6 fdde:4e79:189f::960/128 scope global noprefixroute 
           valid_lft forever preferred_lft forever</broadcast,multicast,up,lower_up> </pre>

3.  Type the weird looking URL to phone browser

    <pre>http://192.168.1.203:8000/
    </pre>

4.  Browse, choose file and download

## Simplicity

<blockuote>“That’s been one of my mantras — focus and simplicity. Simple can be harder than complex: You have to work hard to get your thinking clean to make it simple. But it’s worth it in the end because once you get there, you can move mountains.”  

Steve Jobs, BusinessWeek, May 25, 1998</blockuote>

It is obvious that while using python HTTP server makes things doable for computer geeks, it's not the way regular can reproduce. The major design goal is simplicity, low number of dependencies and easy of use. Qrget is standalone binary, which prints QR Code to be consumed by phone and makes files or directories available using regular HTTP protocol. The only one assumption is that the computer is connected to the same **wireless** network.

Interested? Visit [qrget's github page](https://github.com/vyskocilm/qrget).
