---
layout: post
title: Binding custom development domains
subtitle: Using dnsmasq on macOS
tags: [mac development]
---

I've been programming for many years, and I always avoided using custom domains on my local machine. For small to medium projects, going to `localhost:3000` wasn't so terrible. But in recent years I've worked with multi-tenancy per subdomain and groups of interconnected applications, each with its own subdomain.

Subdomains make those environments more intuitive, so this tutorial explains my approach using `/etc/resolver` and `dnsmasq`. It is more flexible than managing every hostname in `/etc/hosts`, so let's do it.

### Overview of the solution

I'm lazy and don't want to manage more than necessary, so why bother with this setup? A few years ago I worked for a US-based company that was decomposing its monolith into smaller applications. Each service and each customer had its own subdomain, and the [Apartment](https://github.com/influitive/apartment) gem switched the database schema for each tenant.

<pre class="mermaid">
flowchart LR
    customer_a["customer-a.example.test"] --> app["APP"]
    customer_b["customer-b.example.test"] --> app
    customer_c["customer-c.example.test"] --> app
    app --> database_a[("customer_a_")]
    app --> database_b[("customer_b_")]
    app --> database_c[("customer_c_")]
</pre>

To test that application as a user, I needed addresses such as `customer-a.example.test:3000` instead of adapting routes around `http://localhost:3000?apartment=customer-a`.

### Preamble: how URLs are structured

The hostname is one part of a URL. DNS resolves that hostname to an IP address; it does not select an application port. For example, in `www.example.com`, `www` is the subdomain, `example` is the domain, and `.com` is the top-level domain.

For our local DNS server, we first tell macOS which queries to send to it, then configure the server to answer them.

### Step zero: decide a domain to point localhost

There are various options, such as `example.internal` or `example.test`, but avoid:

- `.local`: macOS uses it for [Bonjour services](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/NetServices/Articles/domainnames.html).
- `.dev`: it is a real top-level domain and browsers require HTTPS for it.

This example uses `.test`, which is [reserved for testing](https://datatracker.ietf.org/doc/html/rfc6761#section-6.2). Our application will be available at `example.test`.

### Step one: how can I set a DNS resolver?

On macOS, `/etc/resolv.conf` is system-managed. We can add custom scoped resolvers under `/etc/resolver` instead.

For reliable scoped resolution, [Homebrew recommends](https://formulae.brew.sh/formula/dnsmasq) binding dnsmasq to a non-localhost loopback alias on port 53. The alias disappears after a restart, and I don't want to type `ifconfig` every time, so we will ask `launchd` to create it during boot:

``` bash
sudo tee /Library/LaunchDaemons/com.example.loopback-alias.plist >/dev/null <<'PLIST'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.example.loopback-alias</string>
  <key>ProgramArguments</key>
  <array>
    <string>/sbin/ifconfig</string>
    <string>lo0</string>
    <string>alias</string>
    <string>10.0.0.1</string>
    <string>netmask</string>
    <string>255.255.255.255</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
</dict>
</plist>
PLIST

sudo chown root:wheel /Library/LaunchDaemons/com.example.loopback-alias.plist
sudo chmod 644 /Library/LaunchDaemons/com.example.loopback-alias.plist
sudo plutil -lint /Library/LaunchDaemons/com.example.loopback-alias.plist
sudo launchctl bootstrap system /Library/LaunchDaemons/com.example.loopback-alias.plist
```

The last command creates the alias now and registers the job for future restarts. You can confirm it with `ifconfig lo0 | grep 'inet 10.0.0.1'`.

Now send all `.test` queries to that address:

``` bash
# Send all .test queries to that address
sudo mkdir -p /etc/resolver
echo 'nameserver 10.0.0.1' | sudo tee /etc/resolver/test >/dev/null
```

To check our configuration we will use `scutil --dns`. The output will look like this:

```
... truncated ...
resolver #4
  domain   : test
  nameserver[0] : 10.0.0.1
  flags    : Request A records, Request AAAA records
  reach    : 0x00030002 (Reachable,Local Address,Directly Reachable Address)

```
That result means macOS has loaded the scoped resolver.

### Step two: process the DNS request

Now we need dnsmasq to answer the request with the loopback IP.

Let's install it:

``` bash
brew install dnsmasq
```

Configure it to listen on the loopback alias at port 53 and resolve every `.test` hostname to `127.0.0.1`:

``` bash
# Create the config folder using Homebrew's prefix on Intel or Apple Silicon
sudo mkdir -p "$(brew --prefix)/etc/dnsmasq.d"

# Add the development configuration
sudo tee "$(brew --prefix)/etc/dnsmasq.d/development.conf" >/dev/null <<'EOL'
listen-address=10.0.0.1
port=53
address=/.test/127.0.0.1
EOL

# Start dnsmasq as root so it can use port 53
sudo brew services start dnsmasq

# reset DNS cache
dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

Verify both dnsmasq and the macOS resolver:

``` bash
dig example.test @10.0.0.1 +short
dscacheutil -q host -a name example.test
```

Both should return `127.0.0.1`. You can now open `http://example.test:3000` or use any subdomain under `.test`. DNS resolves the hostname, but your application or reverse proxy still decides which port serves the request.
