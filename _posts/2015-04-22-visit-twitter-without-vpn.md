---
layout:     post
title:      Visit twitter inside firewall without VPN
date:       2015-04-22 19:45
summary:    Twitter is similar with youtube, they both are blocked by firewall via similar filter mechanism, based on the analysis in previous article, I will introduce some ways to visit twitter.
categories: network firewall
---

Like [youtube](/network/firewall/2015/04/03/access-youtube-without-vpn/) and [google](/network/firewall/2015/03/02/access-google-without-vpn/), twitter is blocked in the same way, if you want to know how firewall filter system works, you can read my previous articles, this article I introduce some ways to walk around firewall, and visit twitter freely.  
The main idea is avoid any DNS packets send out for all blocked sites, when visit twitter, we should find out the IP addresses of blocked sites servers, which reachable, and append them into host file.  


how can we know all the blocked sites ?  
we can find them out following steps below:  
1)  open capture tool, like warshark, or sniffer, and start to capture packets  
2)  input twitter.com into your browser  
3)  when browser notify connection time-out or connection reset, stop capturing  
4)  packet analyze, we need to check all the DNS reply, and the connection establish between our device and site's server IP address in DNS reply, if that server never answer our device's request, or connection reset by server, which TTL value is obvious different with other packets in the same connection, they will be the sites blocked by firewall.    

After knowing all the blocked sites, the next step we need to do is find out valid IP addresses for all blocked sites, we can do it via ping these blocked site outside firewall, like online ping website in other countries, or iterate these sites's IP pool.  

For twitter, there are at least two sites, twitter.com and domain with suffix twimg.com, which server store images, like pbs.twimg.com, we need to find out these two sites's IP addresses, and it seems not like google, all services can visit via a single server, in twitter, text and image store at different servers, so we should find them all, or web page won't display normally.    

The last step is append IP address and domain name into host file, flush DNS cache, and enjoy.  

![twitter page](/images/twitter.png)
<center><small>Figure 1: twitter snapshot</small></center>  

