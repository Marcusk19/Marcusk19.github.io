---
title: "How I Blocked Ads on My Home Network"
date: 2021-12-14T08:47:11+01:00
draft: false
---
I’m not a fan of seeing ads everywhere, and even find some of them intrusive. When I discovered Pi-hole, a dns level ad-blocker that supposedly could block ads for all devices on my home network, I was instantly intrigued. I immediately bought an affordable Raspberry Pi Zero W with the end goal of having my own pi-hole on my network.

## Setting up the Pi

To begin, I first had to flash an iso image for my desired operating system to my Pi.

The [official documentation](https://www.raspberrypi.com/documentation/computers/getting-started.html) for getting started was very useful and basically my main source of information when setting up everything for the first time. Basically, I used the [Raspberry Pi Imager](https://www.raspberrypi.com/software/) and flashed the standard version of Raspbian to my SD card as the OS for the Pi.

While there are better practices for setting up a Pi for remote access, for simplicity’s sake I booted the Pi with an external monitor and keyboard attached, went through the initial setup, connected to my network, and enabled ssh. After that had been taken care of, I could detach my external peripherals from the Pi and proceed to do the remaining steps from the terminal of my local machine over ssh.

## Installing Pi-hole

While I can’t remember exactly what online resource I followed to install Pi-hole, [this guide from Tom’s Hardware](https://www.tomshardware.com/how-to/set-up-pi-hole-raspberry-pi) shows pretty much the steps I had to take.

Summarizing the most important steps, I had to first run a curl command to get an installer script and run it.

```
curl -sSL https://install.pi-hole.net | bash
```

It was then a matter of going through the installation prompts (in my case I used wlan0 as the interface for Pi-hole since the Pi Zero W doesn’t have an ethernet port). After that I had my admin portal accessible from my browser and I was ready to go!

One other important step I took, which is also mentioned in the guide, was changing the web portal admin password to something easier for me to remember by entering this command on the terminal:

```
pihole -a -p
```

![](https://marcuskok.tech/wp-content/uploads/2022/01/Screen-Shot-2022-01-21-at-10.51.09-PM-1024x323.png)

Screenshot of the admin dashboard

## How Pi-hole works

A DNS server translates hostnames to IP addresses. In other words, when someone accesses a website (www.example.com) the DNS will translate that domain name into a computer-friendly IP address (192.168.1.0). DNS means that end users don’t have to memorize IP addresses and can instead use easier to remember domain names. In summary, when a user enters www.example.com into their web browser, DNS is what allows the browser to locate and display information from the correct website.

When you set up your router to use Pi-hole as the primary DNS server, your device first goes to Pi-hole when it looks up the address for a host name. If the address belongs to an ad or tracker Pi-hole will block the request, preventing it from reaching your device. If the address isn’t an ad or tracker the lookup will be performed as normal on a DNS server of your choice (e.g. OpenDNS, GoogleDNS, Cloudflare, etc). Basically, Pi-hole acts as a filter on your network for ads.

![](https://marcusk.000webhostapp.com/wp-content/uploads/2021/12/pi-hole.drawio.png)

## Conclusion

After setting up everything I began to test what Pi-hole could block on my network. Unfortunately, ads on YouTube still showed up from my phone and after looking into it more, I learned that it’s because YouTube typically serves ads from the same domain as videos. Pi-hole can’t block everything, but I found that it can block quite a lot. For example, mobile games that typically spam you with banner ads on the edges of the screen had their ads blocked on my network. Convenient because as far as I know, Pi-hole would be the only way I could get rid of ads like that other than paying the app to remove them.

At the end of the day, this was a fun project that only took me a couple hours.I also feel accomplished that I’ve somewhat improved the quality of my home network and can happily browse away without worrying about the ads.