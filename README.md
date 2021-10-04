# The IPocalypse is nigh !

#### A HOWTO for using fewer IPv4 addresses on a FreeBSD server

&nbsp;

For many years, I've had a personal server that mostly does mail and webservers for me and a few friends. It lives in a rack at a colo, its latest incarnation is at Hetzner in Helsinki.  I used to just get a bunch of IP-addresses to go along with it, so some of my projects and some of my friends could have their own 'jail' (FreeBSD's lightweight virtual enironment) with its own IP address.

As I looked at prices for the most recent move (at € 40 per month, the new place is significantly cheaper), I discovered that my regular /28 allotment (14 usable addresses) would cost me another € 32,37 per month as well as a € 361,76 setup fee. I could probably make do with a /29 (6 addresses) if I kill off some jails that are not used frequently anymore, but the other thing my new provider had on its pricing page is the expectation that cost for IPv4 will continue to rise exponentially.

So it made sense to really limit my use of IPv4. After some tinkering I've arrived at a new setup in which my server only has two IPv4 addresses, one for my mail server and one for everything else. Jails that need to be reached from the outside world have an IPv6 address so that they only need port forwarding to be reachable from places or machines that do not have IPv6.

&nbsp;

This HOWTO will show how I set up a FreeBSD server with jails such that it uses NAT for IPv4, with all the jails having only private IPv4. Ports from the host can be forwarded to individual jails to allow access to SSH and other services. No rocket science is involved and this is mostly straightforward stuff. It's written down here in hopes that it helps some people, because some of it can be a bit tedious and time-consuming to figure out. 

To deal with web servers (which all need to be reached at ports 80 (http) and 443 (https), I describe a convenient Apache reverse proxy setup in its own jail, and the management script I wrote to make things super-easy.

The assumption is that you have some basic knowledge of FreeBSD and know how use ezjail to set up jails.

My setup uses FreeBSD (13.0 at the time of writing this), and ezjail for management of the jails, but with some adaptations the reverse proxy setup should be usable on other systems as well.

&nbsp;

## WWW reverse proxy

When you share an IPv4 address between multiple hosts that are providing services, the immediate issue is that some services are offered on fixed port numbers. Nobody likes URLs that point to different ports (such as `https://rop.nl:8080/test.html`). So we need to share the port 80 (http) and 443 (https) between the various jails that are providing web services.

In the very, very early days of the web, each webserver needed its own IP-address because HTTP 1.0 could also ask for the part of the URL after the host part. In HTTP 1.1 a 'Host' header was introduced that allowed browsers to specify which host they meant. It took longer for multiple https sites to be able to share an IP address, as this involved client and server switching SSL-certificates mid-connection.

Today, we can have a web server that 'picks up the phone', looks at the 'Host' header, connects to another webserver and passes the connection along. This is called a 'reverse proxy', and web server software usually comes with modules that allow for this to be set up.

### https, SSL, certificates

There are ways to have the certificates live on the backend servers, but our proxy obtains, stores and periodically renews all SSL-certificates for the websites that lie behind it. It forces all http (port 80) connections to reconect using https and is the endpoint for incoming SSL connections. It then turns around and uses http to connect to the backend servers in the various jails. This has a number of properties:

* The backend servers can be simpler: no need to duplicate site configurations in config files, set up certbot and renewal, no worries that one of my jails hasn't renewed its certificates.

* The backend connections use http, meaning they are unencrypted. So backend servers should be local to the host and not traverse the internet.

* Certificates can't be stolen from the backend jails (which may have all sorts of vulnerable scripts running). Conversely, the wwwproxy jail better not get compromised as all certificates are there.

* More obscure: Backend servers do not get any information about client certificates, so if anything you serve uses client certificates, that'll probably break. Maybe this can be fixed with the client certs on the proxy and then passing on certain headers, but I don't use client certs so I haven't played with that. If you do need that and get it working, please tell me.

&nbsp;

## Setting things up

### On the host system

#### Basic IP setup

Here's the part of my `/etc/rc.conf` that sets up my IP:

```
# IPv4 - external
ifconfig_em0="inet 95.216.35.119/26"           # reverse: jantje.rop.nl
defaultrouter="95.216.35.65"

# IPv4 - internal
cloned_interfaces="lo42"
gateway_enable="YES"

# IPv6
ipv6_enable="YES"
ifconfig_em0_ipv6="inet6 2a01:4f9:2a:2388::1/64"
ipv6_defaultrouter="fe80::1%em0"
```

Jails when they start bring up their own IP aliases, so their IPs do not need to in `rc.conf`. Mine use an 10.42.1.x IPv4 address, plus an IPv6 address if I want them to reachable directly using IPv6. The IPv6 addresses I use for jails are part of my assigned IPv6 /64 which is `2a01:4f9:2a:2388`. I add `:1xxx:`, a one plus all 3 digits of the last octet of the private IP-address as decimal, just so that I don't have to remember two addresses for the same jail. So a jail that has `10.42.1.2` as it's IPv4 address on the `lo42` interface gets `2a01:4f9:2a:2388:1002::1` as IPv6 address like so:

```
ezjail-admin create wwwproxy "lo42|10.42.1.2,em0|2a01:4f9:2a:2388:1002::1"
```

You can use ezjail 'flavours' to make sure your jails already come with the packages you need as well as a correct `/etc/resolv.conf` and anything else you need.

#### NAT, port forwarding

For these jails to have working IPv4 connectivity with the outisde world, we need NAT. NAT, port forwarding and firewall are done with PF. To use it, add to `/etc/rc.conf`:

```
pf_enable="YES"
```

and make sure your pf.conf has:

```
gateway_enable="YES"
```

Now we need to create a config for PF, and with remote servers this is bit of a scary thing, since it's very easy to lock yourself out of your server until you use some kind of remote console to get back in. I wrote [pfedit](https://github.com/ropg/pfedit), a script to edit your firewall config in a way that makes it much harder to lock yourself out. 

Below is an example of a complete `/etc/pf.conf`, a shortened version of what's on my server now:

```
ext_if="em0"
nat_if="lo42"

IP_PUB="95.216.35.119"
NET_NAT="10.42.1.0/24"

scrub in all


wwwproxy="10.42.1.2"
rdr pass on $ext_if proto tcp from any to $IP_PUB port {80,443} -> $wwwproxy

timezoned="10.42.1.3"
rdr pass on $ext_if proto udp from any to $IP_PUB port 2342 -> $timezoned

ns="10.42.1.4"
rdr pass on $ext_if proto udp from any to $IP_PUB port 53 -> $ns
rdr pass on $ext_if proto tcp from any to $IP_PUB port 53 -> $ns

johan="10.42.1.8"
rdr pass on $ext_if proto tcp from any to $IP_PUB port 11008 -> $johan port 22

# nat all jail traffic
nat pass on $ext_if from $NET_NAT to any -> $IP_PUB

mail="95.216.35.78"
mail6="2a01:4f9:2a:2388:f00::1"
mailports="{25,443,465,466,993,995}"
pass in quick on $ext_if proto tcp from any to $mail port $mailports
pass in quick on $ext_if proto tcp from any to $mail6 port $mailports
block in on $ext_if proto tcp from any to $mail 


# passing all other traffic

pass out
pass in
```

As you can see:

* TCP ports 80 and 443 get forwarded to the `wwwproxy` jail.

* I run my own nameserver, so port 53, both TCP and UDP, are forwarded to my nameserver jail.

* I am offering a special service on UDP port 2342, which is passed on to the 'timezoned' jail. This is the timezone service for my project [ezTime](https://github.com/ropg/ezTime).

* I have a jail called 'johan' that has a special port forward (11008) on the host so that Johan can also reach his jail's SSH port if he doesn't have IPv6.

* `mail` is a special jail in that it has a real IPv4 address and does not use NAT. The firewall makes sure only the publicly accessibe ports go there so that it can use various ports for internal services such spam filtering etc. (Jails do not have a `localhost`.)

If you're going to follow this HOWTO, now is a good time to create your own modified version of this `pf.conf` and start pf with `service pf start`.

&nbsp;

### Setting up the wwwproxy jail

Let's create a wwwproxy jail, start it and log in. Naturally you'd use your own IPv6 prefix and not this one, because it's mine.

```
ezjail-admin create wwwproxy "lo42|10.42.1.2,em0|2a01:4f9:2a:2388:1002::1"
ezjail-admin console -f wwwproxy
```

*(The `-f` starts a jail if it isn't running yet.)*

Now install the Apache webserver and Let's Encrypt's certbot:

```
pkg install apache24 py38-certbot
```

Copy the configuration files from this repository to the Apache config `Includes` directory:

```
git clone https://github.com/ropg/ipocalypse
cd ipocalyspe
cp 000-reverse-proxy.conf 010-log.disabled /usr/local/etc/apache24/Includes
```

Put the `domain` script somewhere in your `$PATH` and make it executable. I haven't set up ssh and do everything in this jail as root after logging in from the host with `ezjail-admin console wwwproxy`, so I use `/root/bin` for this.

```
cp domain /root/bin
chmod a+x /root/bin/domain
```

Next, edit `/root/bin/domain` to insert your e-mail address so Let's Encrypt can get in touch if there's something you need to know about your certificates.

Then set up certificate renewal by adding these lnes to `/etc/periodic.conf`:

```
weekly_certbot_enable="YES"
weekly_certbot_post_hook="/usr/local/sbin/apachectl reload"
```

### Logging

During normal operation, you probably want logging disabled: the backend servers for the sites themselves can log what they need, and logging here would only log every HTTP request a second time. So comment out these following lines from `/usr/local/etc/apache24/httpd.conf` by placing a `#` in from of them:

```
LoadModule log_config_module libexec/apache24/mod_log_config.so
```

```
ErrorLog "/var/log/httpd-error.log"
```

While you're debugging this proxy setup, you can use `domain log` (see below) to temporarily get access and errors logged to the screen.

### Turn on Apache

Add to `rc.conf`:

```
apache24_enable="YES"
```

If all is well you can start Apache and you're ready to use your proxy:

```
[root@wwwproxy ~]# apachectl start
Performing sanity check on apache24 configuration:
Syntax OK
Starting apache24.
```

&nbsp;

## On the backend webserver jails

The backend servers don't need to use Apache, they can also run nginx or any other webserver software. They don't need to do anything but know about a given domain and serve it as an http site on port 80. There's a few things that can make things better though.

*(All of the below assumes Apache 2.4 and PHP 7 on the backend, but equivalent solutions exist for other webservers and PHP versions.)*

### No listening on direct IPv6 to jail

If your jails are reachable directly on IPv6, you need to make sure that the webserver does not listen there and that domains that are used in URLs have their `CNAME` (or `A` and `AAAA`) point to the `wwwproxy` jail and not to the backend jail. Otherwise, people might connect directly and https wouldn't work. Or they would be able to connect with http when that's not what you want. With Apache, in `httpd.conf`, simply replace `Listen 80` with `Listen 0.0.0.0:80` to listen only on IPv4.

### Real IP of request, not the proxy

The server logs and any authentication modules see the IP of your proxy as the remote IP. You can make it so these see the real requester's IP instead. If you use Apache on the backend, add the following to its `httpd.conf`:

```
LoadModule remoteip_module libexec/apache24/mod_remoteip.so
RemoteIPHeader X-Forwarded-For
RemoteIPTrustedProxy 95.216.35.119 10.42.1.2
```

### PHP's `$_SERVER['HTTPS']`

If you use PHP, some software will get confused if the browser claims to be using https while the connection comes in on http. To remedy this for all PHP software on a backend server, create a file called `/usr/local/etc/apache24/php-https.php` that contains:

```
<?php $_SERVER['HTTPS'] = 'on' ?>
```

Then add the following to `httpd.conf`:

```
<IfModule php7_module>
    php_value "auto_prepend_file" "/usr/local/etc/apache24/php-https.php"
</IfModule>
```

&nbsp;

## Using the proxy

### DNS from the outside world

To be able to get a Let's Encrypt certificate for a domain, you have to prove that you can place files on a website for that domain. Everything to do with setting things up is automated by our script 'domain', but for all that to work the domain has to point to the proxy. My server is reachable via two IP addresses, and I have set up my nameserver such that `wwwproxy.rop.nl` has an `A` (IPv4) record that says `95.216.35.65` (the IPv4 address of the host) and an `AAAA` (IPv6) record that says `2a01:4f9:2a:2388:1002` (the direct IPv6 address for the wwwproxy jail). Any domains that need to be served can then simply have a `CNAME` that says `wwwproxy.rop.nl`. 

*(Some providers do not allow to create a CNAME for an entire domain, only for subdomains. In that case, just point the `AAAA` and `A` records to your wwwproxy instead.)*

&nbsp;

## `domain` -  the script that makes things easy

Assuming you have built all of the above, now is the time to tell your proxy what domains to proxy for and what jails they live on. I wrote a script to do all this, called `domain`. There are a number of sub-commands:

```
[root@wwwproxy ~]# domain
Usage: domain default <domain>
       domain proxy <domain> <server>
       domain forward <domain> <url>
       domain list [<selector>]
       domain delete <domain>
       domain edit <domain>
       domain log
```

### Default certificate

Our proxy will present a certificate as soon as an SSL connection is made. Because it doesn't know yet what website the other side wants to see, it presents the certificate of the first virtual host. To avoid the confusion of a random ceryificate of one of the websites being presented, we use `domain` to create a special file called `020-default.conf` in the `Includes` directory that we can be sure will be loaded before any of the other domains because they are loaded in alphabetical order. I used the name of the proxy jail:

```
[root@wwwproxy ~]# domain default wwwproxy.rop.nl
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Requesting a certificate for wwwproxy.rop.nl

Successfully received certificate.
Certificate is saved at: /usr/local/etc/letsencrypt/live/wwwproxy.rop.nl/fullchain.pem
Key is saved at:         /usr/local/etc/letsencrypt/live/wwwproxy.rop.nl/privkey.pem
This certificate expires on 2021-12-29.
These files will be updated when the certificate renews.

NEXT STEPS:
- The certificate will need to be renewed before it expires. Certbot can automatically
  renew the certificate in the background, but you may need to take steps to enable that
  functionality. See https://certbot.org/renewal-setup for instructions.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Certicate obtained.
Performing sanity check on apache24 configuration:
Syntax OK
Performing sanity check on apache24 configuration:
Syntax OK
Performing a graceful restart
```

### Putting the backend jails in `/etc/hosts`

I put any jails that offer web services in `/etc/hosts` in my `wwwproxy` jail. It's easier to see what goes where and I don't have to remember the IP numbers that way. So if jail `johan` offers web stuff, I add:

```
10.42.1.8 johan
```

### Setting up a proxy

I have a jail called `web1`, and it serves a number of my domains, including  `rop.nl`. To set it up, I make sure rop.nl is a CNAME for `wwwproxy.rop.nl`, add the internal IP for `web1` to the `/etc/hosts` file on `wwwproxy` and use `domain` to set it up:

```
[root@wwwproxy ~]# domain proxy rop.nl web1
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Requesting a certificate for rop.nl

Successfully received certificate.
Certificate is saved at: /usr/local/etc/letsencrypt/live/rop.nl/fullchain.pem
Key is saved at:         /usr/local/etc/letsencrypt/live/rop.nl/privkey.pem
This certificate expires on 2021-12-29.
These files will be updated when the certificate renews.

NEXT STEPS:
- The certificate will need to be renewed before it expires. Certbot can automatically
  renew the certificate in the background, but you may need to take steps to enable that
  functionality. See https://certbot.org/renewal-setup for instructions.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Certicate obtained.
Performing sanity check on apache24 configuration:
Syntax OK
Performing sanity check on apache24 configuration:
Syntax OK
Performing a graceful restart
```

### `www.` subdomains

Some backend servers might accept the `www.` domian as a ServerAlias. To make sure this still works I could have made this addtional names on the same certificate, but the logic was more complicated than just creating a new certificate with that name and forwarding it to the same backend jail. So just repeat the process for any subdomains after you made sure their DNS points to the proxy:

```
domain proxy www.rop.nl web1
```

### Forwarding domains

While it's made for proxying, you can also have `domain` create a config file to have the domain forwarded somwehere else. (Reverse proxy is when the web server gets the data from a backend server and the browser doesn't know about it, forward is when the browser is told to go somewhere else and show that in the URL bar instead.)

I used to have a blog at `rop.gonggri.jp`. But I haven`t blogged in years. So I decided to foreward that subdomain to my twitter account instead:

```
domain forward rop.gonggri.jp https://twitter.com/rop_g
```

If you want to forward to a URL that matches the entire request on the old URL, just add `\$1`, like in:

```
domain forward somedomain.com https://otherdomain.com\$1
```

### Other `domain` sub-commands

#### list

```
[root@wwwproxy ~]# domain list
domain default wwwproxy.rop.nl
domain forward rop.gonggri.jp https://twitter.com/rop_g
domain proxy rop.nl web1
```

As you can see the output of this command is also what you would need to execute to configure the same proxy again.

#### delete

```
domain delete somedomain.com
```

Will ask you whether you want to revoke the certificates as well and then delete the config file for the domain and reload the server.

#### edit

```
domain edit somedomain.com
```

Will show the `somedomain.com.conf` file that `domain` has created in the editor set in `$EDITOR` for you to edit. Reloads the server if you made any changes. You probably won't need to edit any files manually.

#### log

```
domain log
```

Reloads the server with access and errors logging to the screen to aid in debugging. If you interrupt with Ctrl-C the log is stopped and the server reloaded again.
