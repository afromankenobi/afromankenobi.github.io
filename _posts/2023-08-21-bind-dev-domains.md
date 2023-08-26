---
layout: post
title: Binding custom development domains
subtitle: Using dnsmasq on MacOS
tags: [mac development]
---

I've been working as a programmer a lot of years and i always avoided the idea of use custom domains to point my own machine. I mean, there are some scenarios where i justified my decision with very fair arguments like overkill feature because in small to mid complex work going to `localhost:3000` wasn't so terrible, but for the latests years i've fighting with issues like multi-tenancy per subdomain, or a graph of interconected applications via subdomain. The idea of subdomains is *intuitive*, makes the environment you're using feel interconnected so i decided to put in a tutorial my approach to work with subdomains in local machines via `/etc/resolvers` combined with `dnsmasq`. If you compare this approach versus the classic usage of `/etc/hosts` file is way more flexible, so let's do it:

### Overview of the solution

I'm a lazy man, and usually i don't want to manage more things than the necessary ones, so why bother with this setup?
Well, a couple of years ago i started working for a US based company that was decomposing their monolith into different subapplications. Each services had their own subdomain, and each customer had their own subdomain. They were switching database schema using [Apartment](https://github.com/influitive/apartment) gem.


``` ascii
┌───────────────────────┐                                 ┌──────────────┐
│                       │                                 │              │
│ customerA.app.example │                     ┌───────────► customerA DB │
│                       ├─────┐               │           │              │
└───────────────────────┘     │               │           └──────────────┘
                              │               │
┌───────────────────────┐     │  ┌─────┬──────┴────┐      ┌──────────────┐
│                       │     │  │     │ Apartment │      │              │
│ customerB.app.example ├─────┼──► APP │  switch   ├──────► customerA DB │
│                       │     │  │     │ mechanism │      │              │
└───────────────────────┘     │  └─────┴──────┬────┘      └──────────────┘
                              │               │
┌───────────────────────┐     │               │           ┌──────────────┐
│                       │     │               │           │              │
│ customerC.app.example │     │               └───────────► customerA DB │
│                       ├─────┘                           │              │
└───────────────────────┘                                 └──────────────┘
```

![MultitenantDiagram](/assets/img/database_multi_tenant.drawio.png)

By the way, we used [this](https://github.com/influitive/apartment/wiki/Testing-Your-Application) approach to test features.
Let's suppose that we need to create an automated onboarding process. A simple sign up form requesting the user data and the company name. Now when the user register into our application we need to:

- Ensure Tenant is created. Our tenant here is generated from the company name.
- Ensure user data is okay
- Ensure user belongs to company.

I will leave that explanation in deep for another post and to try to go to the point: What happen if i want to test like a user my application? How can i write a subdomain if i need to access my application on the port 3000?

That's what we are gonna resolve here.

| Of course that before this i just adapt my routes to do something like `http://localhost:3000?apartment=company_a`, but it's ugly.

### Preamble: how URLs are structured

This is so related to our issue that we need to touch it a little bit. I'm not an expert, but as far as this post involves, we need to understand that URLs are composed by the following parts:

![URL_STRUCTURE](https://blog.syncitgroup.com/wp-content/uploads/2019/12/url-structure-1024x411.jpg)

DNS records are part of the URL, and are composed by a subdomain (optional), a domain, and a TLD. In example `www.asdf.com`, `www` is the subdomain, `asdf` is the domain, and `.com` is the TLD. When you browse to that URL your computer do is to lookup the DNS:

                                           ┌───────────────────────────┐
                                      ┌───►│   DNS Recursive Resolver  │
                                      │    └───────────────────────────┘
                                      │
                                      │    ┌───────────────────────────┐
┌────────────────┐                    ├───►│   Root Nameservers        │
│                │   ┌────────────────┤    └───────────────────────────┘
│ www.google.com ├──►│DNS local cache │
│                │   └────────────────┤    ┌───────────────────────────┐
└────────────────┘                    ├───►│  TLD Nameserver           │
                                      │    └───────────────────────────┘
                                      │
                                      │    ┌───────────────────────────┐
                                      └───►│  Authoritative Nameserver │
                                           └───────────────────────────┘


When we setup a local DNS server, first we need to tell our computer to not to lookup for the DNS outside. Then we need to say it how to define the different parts of the DNS.

### Step zero: decide a domain to point localhost

Some people use things like `app.internal`, `company.applicationname.test`. There are lots of options in the wild software world to use. You can use whatever you like, except:

- local domain: In MacOS they're used to connect with Bounjour services. [This](https://support.apple.com/en-us/HT203136) article explain why. You can use `example.local` too if you want
- dev domain: .dev domains are part of top-level domain (TLD) and browsers start to require certificates for those. [Here](https://medium.engineering/use-a-dev-domain-not-anymore-95219778e6fd) an interesting article about that.

In this example we will use `test` domains. So, to access our applications we will use `fakedomain.test` so we can emulate a realURL.

### Step one: how can i set a DNS resolver?

In UNIX-like system we can configure how DNS queries are executed configuring `/etc/resolv.conf`. That file contains the rules for DNS query resolution, but it a system managed file, so we need to use a directory that can contain all our custom configuration: `/etc/resolver`.

``` bash
# If you don't have the folder, create it
$ mkdir -p /etc/resolver
$ touch /etc/resolver/test
```

Then, add the following line:

``` conf
nameserver 127.0.0.1
```

We are configuring our system to resolve all queries for domains ending in `test` to the nameserver 127.0.0.1. So, from now on, each time we do `ping whatever.test` it will look for the definition in our local machine.

To check our configuration we will use `scutil --dns`. The output will look like this:

```
... truncated ...
resolver #4
  domain   : test
  nameserver[0] : 127.0.0.1
  flags    : Request A records, Request AAAA records
  reach    : 0x00030002 (Reachable,Local Address,Directly Reachable Address)

```
That result means everything is working fine.

### Step two: processing the DNS request.

It's not enough to resolve a DNS to our local machine. Now we need to process that request and return an adequate IP for the host. That's where DNSMasq is the main role player.

Let's install it:

``` bash
brew install dnsmasq

# start it
sudo brew services start dnsmasq
```

Then, we will resolve DNS request for `test` domain:

``` bash
# create the configs folder in case it doesn't exists
sudo mkdir -p /opt/homebrew/etc/dnsmasq.d

# add the configuration into /opt/homebrew/etc/dnsmasq.d/development.conf
sudo tee /opt/homebrew/etc/dnsmasq.d/development.conf >/dev/null << EOL
address=/.test/127.0.0.1
EOL

# restart DNSMasq
sudo brew services restart dnsmasq

# reset DNS cache
dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

That should be enough to 
