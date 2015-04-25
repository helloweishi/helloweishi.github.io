---
layout:     post
title:      Defense RST attack from firewall
date:       2015-04-25 19:54
summary:    In keywords filtering system. firewall will fire RST, which make connection broken, if there are some sensitive information found, this article I introduce how can we defense this attack at Linux.
categories: network firewall
---

Keywords filtering system is one of filtering mechanism in firewall, it can make the connection broken immediately, and the service we visit will stop, one simple example is we search something via Google, browser display connection reset, right after we input the search keywords.  


In order to explain how we can fix it up, I introduce some data structure first:  

>struct proto tcp\_prot = {  
>        .name                   = "TCP",  
>        .owner                  = THIS_MODULE,  
>        .close                  = tcp\_close,  
>        .connect                = tcp\_v4_connect,  
>        .disconnect             = tcp\_disconnect,  
>        .accept                 = inet\_csk\_accept,  
>        .ioctl                  = tcp\_ioctl,  
>        .init                   = tcp\_v4\_init\_sock,  
>        .destroy                = tcp\_v4\_destroy\_sock,  
>        .shutdown               = tcp\_shutdown,  
>        .setsockopt             = tcp\_setsockopt,  
>        .getsockopt             = tcp\_getsockopt,  
>        .recvmsg                = tcp\_recvmsg,  
>        .sendmsg                = tcp\_sendmsg,  
>        .sendpage               = tcp\_sendpage,  
>        .backlog\_rcv            = tcp\_v4\_do\_rcv,  
>        .release\_cb             = tcp\_release\_cb,  
>        .hash                   = inet\_hash,  
>        .unhash                 = inet\_unhash,  
>        .get\_port               = inet\_csk\_get\_port,  
>        .enter\_memory\_pressure  = tcp\_enter\_memory\_pressure,  
>        .stream\_memory\_free     = tcp\_stream\_memory\_free,  
>        .sockets\_allocated      = &tcp\_sockets\_allocated,  
>        .orphan\_count           = &tcp\_orphan\_count,  
>        .memory\_allocated       = &tcp\_memory\_allocated,  
>        .memory\_pressure        = &tcp\_memory\_pressure,  
>        .sysctl\_mem             = sysctl\_tcp\_mem,  
>        .sysctl\_wmem            = sysctl\_tcp\_wmem,  
>        .sysctl\_rmem            = sysctl\_tcp\_rmem,  
>        .max\_header             = MAX\_TCP\_HEADER,  
>        .obj\_size               = sizeof(struct tcp\_sock),  
>        .slab\_flags             = SLAB\_DESTROY\_BY\_RCU,  
>        .twsk\_prot              = &tcp\_timewait\_sock\_ops,  
>        .rsk\_prot               = &tcp\_request\_sock\_ops,  
>        .hashinfo             = &tcp\_hashinfo,  
>        .no\_autobind            = true,  
>\#ifdef CONFIG\_COMPAT  
>        .compat\_setsockopt      = compat\_tcp\_setsockopt,  
>        .compat\_getsockopt      = compat\_tcp\_getsockopt,  
>\#endif  
>\#ifdef CONFIG\_MEMCG\_KMEM  
>        .init\_cgroup            = tcp\_init\_cgroup,  
>        .destroy\_cgroup         = tcp\_destroy\_cgroup,  
>        .proto\_cgroup           = tcp\_proto\_cgroup,  
>\#endif  
>};  


Struct proto tcp\_prot is the central structure of TCP protocol in Linux kernel, which define in file net/ipv4/tcp\_ipv4.c, all the connection control and packets in/out are hanled by HOOK functions in this structure, like HOOK function backlog\_rcv receive packets from layer 3, and deliver them to upper layer.


>struct inet_sock {  
>	/* sk and pinet6 has to be the first two members of inet_sock */  
>	struct sock		sk;  
>\#if IS_ENABLED(CONFIG\_IPV6)  
>	struct ipv6_pinfo	*pinet6;  
>\#endif  
>	/* Socket demultiplex comparisons on incoming packets. */  
>\#define inet_daddr		sk.\_\_sk_common.skc\_daddr  
>\#define inet_rcv_saddr		sk.\_\_sk\_common.skc\_rcv\_saddr  
>\#define inet_dport		sk.\_\_sk\_common.skc\_dport  
>\#define inet_num		sk.\_\_sk\_common.skc\_num  
>   
>	\_\_be32			inet\_saddr;  
>	\_\_s16			uc\_ttl;  
>	\_\_u16			cmsg\_flags;  
>	\_\_be16			inet\_sport;  
>	\_\_u16			inet\_id;  
>  
>	struct ip_options_rcu \_\_rcu	*inet\_opt;  
>	int			rx_dst\_ifindex;  
>	\_\_u8			tos;  
>	\_\_u8			min\_ttl;  
>	\_\_u8			mc\_ttl;  
>	\_\_u8			pmtudisc;  
>	\_\_u8			recverr:1,  
>				is_icsk:1,  
>				freebind:1,  
>				hdrincl:1,  
>				mc\_loop:1,  
>				transparent:1,  
>				mc\_all:1,  
>				nodefrag:1;  
>	\_\_u8			rcv\_tos;  
>	\_\_u8			convert\_csum;  
>	int			uc\_index;  
>	int			mc\_index;  
>	\_\_be32			mc\_addr;  
>	struct ip\_mc\_socklist \_\_rcu	*mc\_list;  
>	struct inet\_cork\_full	cork;  
>};    


Struct inet\_sock maintains network elements of connection established, like IP addresses and ports of foreign and local, TOS value, outgoing device index, etc, and it is embedded in struct sock sk, which is the central structure of socket, and which controls and maintains all the states of connection, we can use inet\_sk(sk) call to get struct inet\_sock information from struct sock. There are two elements related to TTL in struct inet\_sock I like to mention here, uc\_ttl is TTL for local machine, TTL in IP header will take its value when send out, if its value is not -1, which is the value when socket init, but we can set its value via ioctl; mc\_ttl is TTL for multicasting, for unicast, there is no use, so we can use it to record TTL value received from peer, which can help us to fixup RST attack, when socket init, mc\_ttl is set to 1.


>code segement of function inet_create, when socket init.  
>	inet->inet\_id = 0;  
>   
>	sock\_init\_data(sock, sk);  
>   
>	sk->sk\_destruct	   = inet_sock_destruct;  
>	sk->sk\_protocol	   = protocol;  
>	sk->sk_backlog\_rcv = sk->sk_prot->backlog\_rcv;  
>   
>	inet->uc\_ttl	= -1;  
>	inet->mc\_loop	= 1;  
>	inet->mc\_ttl	= 1;  
>	inet->mc\_all	= 1;  
>	inet->mc\_index	= 0;  
>	inet->mc\_list	= NULL;  
>	inet->rcv\_tos	= 0;   


Ok, let's start to dig code to find out how TCP socket works in Linux kernel, to simplify the case, I only talk about receiving diretion.

![TCP FLOW](/images/tcpflow.png)
<center><small>Figure 1: TCP receive flow</small></center>  

when socket start, we are client, we start the connection, after socket init, 3 way hand shake start:  
HOOK function connect, for TCP tcp\_v4\_connect, send SYN to server, and set TCP socket state to TCP\_SYN\_SENT;  
Server reply SYN ACK, it is handled by HOOK function backlog\_rcv, tcp\_v4\_do\_rcv in TCP, it calls tcp\_rcv\_state_process to deal with state TCP\_SYN\_SENT;  
tcp\_synsent\_state\_process called to handle SYN ACK from server, it send SYN ACK back to server, and set TCP socket state to TCP\_ESTABLISHED, if TCP flags server send are not SYN ACK, like RST, it will deal with in different way.  


>TTL record in tcp\_synsent\_state\_process  
> we can record TTL value from server here after tcp state to TCP\_ESTABLISHED  
>\# code segement  
> inet\_sk(sk)->mc\_ttl = ip_hdr(skb)->ttl;  
>   


after connection established, HOOK backblog\_rcv will handle stream in fast path under the condition of flag in TCP header match flag in previous packet, it calls tcp\_rcv\_established, if there is one packet with flag RST recevie, it will obvious make flag unmatch with previous one, and packet will go to slow path, tcp\_validate\_incoming will be called to handled the exception, which define in net/ipv4/tcp\_input.c.  

In tcp\_validate\_incoming, flags checking plays significant role, including RST flag checking.  


>code segment of function tcp\_validate\_incoming  
> 	/* Step 2: check RST bit */  
>	if (th->rst) {  
>		/* RFC 5961 3.2 :  
>		 * If sequence number exactly matches RCV.NXT, then  
>		 *     RESET the connection  
>		 * else  
>		 *     Send a challenge ACK  
>		 */  
>		if (TCP\_SKB\_CB(skb)->seq == tp->rcv\_nxt)  
>			tcp\_reset(sk);  
>		else  
>			tcp\_send\_challenge\_ack(sk, skb);  
>		goto discard;  
>	}  
>   


By default, tcp\_reset will be called to terminate the connection, if RST flag set.  
Defense attack fixup:


>code segment of function tcp\_validate\_incoming  
> 	/* Step 2: check RST bit */  
>	if (th->rst) {  
>		/* RFC 5961 3.2 :  
>		 * If sequence number exactly matches RCV.NXT, then  
>		 *     RESET the connection  
>		 * else  
>		 *     Send a challenge ACK  
>		 */  
>       /* if TTL difference between RST and SYN packet received from peer more than 4, ignore RST, it is potential attack */  
>       if (inet\_sk(sk)->mc\_ttl > 1 && abs(ip\_hdr(skb)->ttl - inet\_sk(sk)->mc\_ttl) > 4)  
>           goto discard;  
>		else if (TCP\_SKB\_CB(skb)->seq == tp->rcv\_nxt)  
>			tcp\_reset(sk);  
>		else  
>			tcp\_send\_challenge\_ack(sk, skb);  
>		goto discard;  
>	}  
>   


Fix me, if you find something wrong in my article, Thanks.







