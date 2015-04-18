---
layout:     post
title:      Visit Google inside firewall without VPN
date:       2015-03-02 12:20:18
summary:    This article introduce how firewall system block google site accessing, and how we can break its access limits.
categories: network firewall
---

I believe many people don't like restrictions, we live in a equal world, we love freedom, but when you want to search something, you find you can't even open that search engine's main page, I believe you must feel bad, what's going on there, it that a firewall system stuck your way ? 

So this article, I like to, from technical point of view, analysis how firewall works, why you request blocked, and how to break it.

Here we go, for a firewall system, it will block your request if it know 

 1.  it knows where you want to request and that destination is in black list
 
 2.  it knows what you want to request and it contains some sensitive information

**For 1)**: when you type "www.google.com" in your browser, your PC/Phone will send DNS query to DNS server if your PC/Phone don't have that domain's IP address in DNS cache, or static configured, so one direct way to know where you want to request is check your DNS query, there are two way to block your request, 

option 1: Drop your DNS query, your DNS request can never reach DNS server;

option 2: Reply you DNS query with wrong IP address, which is called DNS poison.

In real appliance, blocking DNS query don't block many requests, because DNS query can never reach firewall, all DNS servers we use by default are inside local ISP, normally, several routers away from your PC/Phone, so your DNS query can never go that far to firewall system, and you can try to configure your device's DNS server to a foreign server, and check if firewall system will block your DNS query, or not.
normally, your device will get some valid IP addresses, and then, your device will send an SYN packet to IP address, which is Google server, and now, firewall system start to work, because there is no server inside, all the rest requests will go through firewall system, so here, it can drop your request directly for any requests to that Google server's IP address, that is why you open your browser, type  "www.google.com", and wait for a long time, and then, your browser display connection timeout.

How firewall system know Google server's IP address? especially, it is not just one, but many, it can know part of it, there are several ways to know that, one way is the corporation of ISP, ISP define a small IP pool of Google domain as DNS replying, firewall system can only block all IP in this pool, it can block all requests inside; another way is we can know who some public IP belong to via whois utility, for big company, it usually have a large IP pool, which we can find out via whois online. So for firewall system, it can not block that small IP pool ISP defined, it need more, and actually, it does. but it can not know all of them, and this will leave some space to us.

**For 2)**: In some period, you find you can open Google's main page, but when you type some search keywords, your browser display connection reset immediately, that is another filter system, it not only check your request destination, but also check your request content, which do via pattern matching. normally, when you request Google service, your device will connect service port 80, which is HTTP request, it is plain text connection, everything you request can be seen by the router of routine, and every time you want to search something, its keywords you typed contain in type GET/POST kind of HTTP packets, firewall system can only check this two kind of packets, it will know what you want to request, and block it if sensitive, actually it is not forbid your request pass through, but send a RST packet to you device actively, which will make the connection between your device and Google server broken.

Why firewall system don't just drop you request packet, which will make your request never arrive Google server, and which you may think it is the simplest way, it will effect network throughput seriously if it does in this way, why? that is due to the design of network mode, it have 7 layers, but in real application, we just consider about 5 layers, which is physical layer, data link layer, network layer, transport layer, application layer from bottom up, IP information is contained in network layer, HTTP content is contained in application layer, for a router in the routine between your device and Google server, if it want to know what your request are, it have decode packets from your device bottom up, layer by layer, less layer it decode, faster packets can pass through, assume the time it take is the same each layer it decode, and it takes 1 second to forward one packet when it just decode one layer, if it decode two layer, it has to take 2 second, and etc, HTTP content is contained in application layer, can you image that ? it will be nightmare. Usually, router just need to decode three layers when forward, that means it just need to know where your request go, don't care what your request are, for some network, they just decode two layers for throughput speed up, like using route protocol MPLS, it also called MPLS network. 

So for keywords filter system, I guess your HTTP request don't pass through firewall directly, firewall is connected at the router, which forward your request packets to outside internet, when your request packets pass through, it will forward twice, one to outside internet, one to firewall, packet forward is pretty fast, it just sacrifice performance a little, or not at all. firewall will decode packets bottom up till application layer, if it find some sensitive keywords, RST packet will be sent to your device, connection broken. We can verify it, first, make sure you don't surf internet via proxy, and then, capture packets when you search some sensitive keywords, from the packets your captured, you can see the time interval between your request packet and RST reply packet is much less than the interval between two continuous packets when connection establish, and after RST packet, Google server's search reply packet arrive, too, and we can check TTL value in IP header, you can find TTL value in RST packet is much different from other packets from Google server. TTL means time to live, when one packet send from a device, it have a initial value, which usually is 128, or 255, every time this packet pass through a device, its value will decrease 1, if its value reach 0, this packet will be drop silently, from TTL value, you can how far it is your device from Google server,  and you can know how far firewall from your device. And the path go to and back from Google server may different, which depends on route protocols, TTL value may have a little difference from time to time, but usually the difference can't reach more than 3 roughly, except the whole internet fluctuate a lot, that seems never happens so far, if it does, it will be a big news.

![TTL](/images/ttl.png)  
<center><small>Figure 1:Time to live snapshot</small></center>


**Time to break restriction**:

From analysis above, I think you already know how firewall system work, so now, we start to break it.

Break restriction 1:

Remember I mentioned above, for a big company like Google, it has a large IP pool, if you try to get Google's IP address via DNS query inside firewall system, it must fail, all the IP addresses you get inside are blocked directly, they are already listed in black list. 

One way to get a valid Google server IP address is try to get it from outside DNS servers, like some DNS servers in some country outside firewall system, you know Google have many servers in different countries, that can help user in different country getting better experience, and DNS server will return DNS request with Google server's IP address in its own country first, maybe you will ask how can we get IP address from outside DNS servers, there are some online ping website, some ping website can ping the domain name you specified, like Google.com, from different countries at the same time, and return some valid IP address to you if ping that domain name succeed, normally, these IP addresses are different, after you get these IP address, you can just input these IP addresses in your browser one by one, if you are lucky, Google search page will show on the screen. Google search page are the default page if you just input IP address in browser, for now, you can just use Google's search service, other services like maps, Gmail, youtube, and etc, you still can't use, which I will talk about it in later article.

Another way to search some valid Google server IP addresses is iterate Google's IP pool, but you need more advanced skills, whois utility can be use to search Google's IP pool, and there are some whois websites you can search at, IP pools is very large, it is pretty hard to ping them by hand,  one by one. There are many packets generator tools online, which can construct ping packets, it can iterate the whole IP pool in short time, after checking IP address ping succeed, you can get some valid IP addresses, or you can write a script via python, shell, and etc, and check the ping successful IP addresses.

Break restriction 2:

For keyword filter system, it can work, that is because the connection between your device and Google server is not encrypted, anyone in the middle can see it, you should use a secure one, which is HTTPS connection, when you open Google website, try to input https://www.google.com, instead of www.google.com, or just google.com.

Access Google via HTTPS, all the context between your device and Google server are encrypted, there is no way to decrypted them in the middle, even firewall system can get, but it has no way to know what it is.

![google](/images/google.png)  
<center><small>Figure 2:google search snapshot</small></center>

Ok, time to stop, if any questions, you can drop me a email, next article I will talk about how to visit youtube inside firewall system.  
Fix me, if you find some errors in my post.

