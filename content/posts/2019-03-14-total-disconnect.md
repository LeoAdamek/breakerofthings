---
layout: post
title: Total Disconnect and the Trouble of Distributed Networks
date: "2019-03-14"
---

# Introduction

Corporations exist to make money, not to help people, but to help themselves. 
This is a fact of any capitalist socity. Many corporations operate responsibly, 
others do not. 
In the past, when a corporation's actions were disagreeable, showing your 
dissastisfaction was as simple as not purchasing the products or services of it 
or any of its subsidiaries.
In the internet age, this isn't as simple as it first seems, in this post we'll 
explore the technical challenges which present themselves when trying to
initiate a complete disconnect from a large internet-based corporation.

# I. The Goals and Requirements

The goal of this project is to implement a "complete disconnect", a barrier
between our own system or network, and that of any company owned by the target
corporation. We wish to implement a total embargo on any data originating from
our network, and transiting to any of theirs.

For this post, we'll concerntrate on Facebook, Inc. and all subsidiaries and
group companies, both complete and partially owned. 

# II. Methods of Control

There are multiple methods to implementing boarder control on a network.
Each has its own advantages and disadvantages, and none of them are adequite to
implement a completely closed boarder on their own.

### Domain Blocking

One of the simplest and most common methods of implementing blocking at a
network level involves modification to how networks are found, through DNS.

By altering the addresses given to systems seeking to discover domains owned by
the target operator, it is possible to either direct the user to a non-existant
address, the local system, or another system which can inform the user that
access to the target network is disabled.

Domain blocking works well for smaller networks with few domain names. However,
it is difficult to block larger networks which operate many domains as no
central registry of domain name ownership exists, nor is it possible to query
a domain registry for a list of domains by owner entity. Domains which are to be
blocked must be manually discovered.

Using recursive domain name server software, 
such as [dnsmasq](http://dnsmasq.org/), we can override the address given for a
given domain.

Using dnsmasq we can implement a rule such as the following:

    address=/facebook.com/10.1.0.205

Which will instruct DNSmasq to return the address `10.1.0.205` for `facebook.com`
or any subdomain of it.

Name-based blocking has its limitations though:

1. Devices, users and programs can easily use a different name resolver to
   bypass the block and retrieve the address of the real network.
2. No central registry of domain names exists, registries are implement at the
   top-level domain. While most registries provide programmatic access to this
   data via the WHOIS protocol, it isn't required and some registries do not
   provide programmatic access.


## Address-based Firewall Blocking

The most effective way to block access to a remote network is to directly block,
reroute or otherwise manipulate traffic destined to it.

In this post we'll be using
[iptables](https://netfilter.org/projects/iptables/index.html) to reject any
connections directed to a network operated by the target organization. 

The first question is how to work out which networks we need to block.
In the case of Facebook itself, this can be done quite easily as Facebook is
large enough that it has its own address allocations which can be queried via 
its Autonomous System Number (ASN), 32934 to get a list of all address ranges
advertised by Facebook on the internet.

We'll use the [Robtex API](https://robtex.com/) to query the address ranges
advertised by Facebook.

````http
GET https://freeapi.robtex.com/asquery/32934
Accept: application/json
````

This will return some JSON data in the following format:

````json
{
    "nets": [
        {
            "n": "31.13.24.0/21",
            "inbgp": 1,
        }
    ]
}
````
We can now collect all the network addresses from this list and generate
firewall rules to block these networks.

First we'll add a new chain to iptables to allow us to change what happens to
traffic matching a filter
All iptables commands must be run as the root user as they manipulate low-level
networking parameters.

    iptables -N FACEBOOK

For each address range, we can add a rule to the built-in `OUTPUT` chain to have
the traffic passed to the new `FACEBOOK` chain:

    iptables -I OUTPUT -d $ADDRESS_RANGE -j FACEBOOK

Finally, we'll add a rule to the `FACEBOOK` chain to reject all traffic
passed to it:

    iptables -I FACEBOOK -j REJECT

Note that this approach is for implementing the rule on an end-user system.
With one simple modification we can implement this rule on an iptables-based
router. Instead of inserting the rules to the `OUTPUT` chain, we can add them to
the `FORWARD` chain.

With the action for the `FACEBOOK` chain set to `REJECT`, the user device will
behave as though the network has been disconnected when trying to connec to any
of the listed addresses. The action can also be set to `DROP` which will cause
the commication to be ignored and the user device will wait until it times out.

By adding all rules to a dedicated chain, we can change the behaviour for all
rules with a single update, and can get statistics about how many connections
have been blocked overall.

## Deep-Packet Inspection

The most effective method of cutting off a remote network is also by far the
most invasive. Deep-packet inspection involves passing all network traffic
through a system which reads the full contents of the data and then determines
weather or not to pass the data on, or to return something else.

This method requires the most investment of time and compute resources, as
the filtering rules will be highly complex, and requires reading and repackaging
all network traffic.

This method is also highly invasive of user privacy as it requires that all of
the user's traffic be inspected in full by another party.

For these reasons, this post won't go into too much detail on the specific
implementation of these rules.


# III. Conslusions

There are problems and challenges with any single method to block access to
specific systems, while maintaining privacy of the end user when accessing
other services.

As networks and systems become more interconnected, and the
scope of ownership within those networks becomes ever more flexible, identifying
the true owner of a remote system you're connecting to becomes ever more
difficult. While this is great for the spirit of openness and neutrality, when a
bad actor found, disconnecting from their networks becomes increasingly more
challenging. This issue is compounded as a few major corporations take ownership
of ever growing segments of the many networks that comprise the internet as it
exists today.

In a future post, I'll talk about some solutions being developed by activists
and volunteers to bring privacy, and decentralized control back to the internet.