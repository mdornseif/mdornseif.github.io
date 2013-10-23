---
layout: post
title: How to weather through DDos on AppEngine (en)
---

{{ page.title }}
================

Google AppEngine should scale on demand and let you handle any workload.
At least this is, what it says on the packaging. Turns out this is not
completely true. This is also a very good thing because probably you don't
want to pay for every workload.

We came across this issue a few monts ago when we got under a Distributed
Denial-of-Service (DDoS) Attack. Thousands of computers arround the world
where requesting millions of pages from our web presence. I'll sum up what
we have done

The Google Infrastructure scaled up our application untill it was running
about 1000 instances and then stopped strating new instances. This resulted
in legitimate users experiencing a very bad freformance and timeouts
for the site. Until that moment we didn't notice anything - the joy of
cloud computing.


Mitigating the Attack - first steps
-----------------------------------

We use the Website als for a lot ot internal tools. So we deployed
and additional version `relief` and pointed all our internal users
to `http://relief.ourapp.appspot.com` and for them the problem was solved,
work in the office could resume.


Google DDoS Protection Service
------------------------------

The Google AppEngine infrastructure provides a [DDoS Protection service](https://developers.google.com/appengine/docs/python/config/dos). This can be used to blacklist
up to 100 IP-Adresses or Networks. Requests from blacklisted IPs
will not reach your application.

At the Appengine Administration Console under Administration -> Blacklist
you can get a list of most active users. We put them in a `ddos.yaml`
redeployed and thought we where all set.

To make this more easy to handle we generated a text file of network names
and a tool to process them into `ddos.yaml`.

    50.10.215.87
    68.45.242.187
    101.12.118.50
    ...
    218.15.223.142

<script src="https://gist.github.com/mdornseif/6682539.js"></script>


Unfortunately it turned out that while `ddos.yaml` has a limit of 1000 entries
we where attacked by far more hosts than 100. So the first step for us was top
get the worst offenders. By downloading the logs from Google and doing some shell
scripting this was easy:

    appcfg.py request_logs --num_days=2 -V production . ./logs.txt
    cat logs.txt  | cut -d ' ' -f 1 | sort | uniq -c | sort -nr | \
       perl -npe 's/.*(\d+) ([\d.]+)/\2 # \1/;' > ddos.txt

So how to now get ddos.txt down to 100 Lines? We started manually aggregating
networks. So

    87.67.81.76 # 7
    87.67.73.148 # 6
    80.187.96.149 # 5
    80.187.108.173 # 7

became

    87.67.0.0/16
    80.187.0.0/16

You might read up on [CIDR](http://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing)
if you don't understand what the slashes mean. It turnt out that we
where over-blocking.

So wh had to to find a way to find the real network sizes of the offending
networks. `whois 87.67.73.148 | grep -E 'inetnum|descr'` gave us the network size,
e.g. `87.67.64.0 - 87.67.95.255`. If you can't convert that to CIDR notation
by yourself, tools like [ip2cidr](http://ip2cidr.com/bulk-ip-to-cidr-converter.php)
help. Enter `87.67.64.0,87.67.95.255` and get `87.67.64.0/19`. So we put that into our
ddos.txt (and regenerated ddos.yaml and uploaded it).

    87.67.64.0/19  # Belgacom
    80.187.0.0/18  # DTAG
    80.187.64.0/19 # DTAG
    80.187.96.0/20 # DTAG

This still was not enough. 100 Entries could still not block all the different
providers we where seeing attacks from. We obviously avoided putting
european IP ranges in the blacklist - because our customers are from there.
We saw that most requests came from Asia - where we practically have no customers.
So "block Asia" was the way to go. We began searching for huge ("Calss-A")
Netblocks being exclusively useed in Asia.

By using `whois 110.0.0.0/8 | grep descr` we could find out that
110.0.0.0/8-125.0.0.0/8 and 79.* to 95.* where exclusively used "overseas".
Adding this huge blocks together with the most active providers into
`ddos.txt` got our app into a usable state again.


In-App country blocking
-----------------------

Obviously this blocking of huge address spaces was a radical approach but
it allowed our website to be accessible again to our users. We where aware
that our `ddos.txt` would need permanent management and supervision.
So we looked for a way to automate blocking by country.

The idea was to do that in our webapp but so early the DDoS requests would
only need minimal resources. Since Google AppEnginge already provides
Geolocation Services at no additional costs this was eary to implement.

<script src="https://gist.github.com/mdornseif/6682867.js"></script>

Appengine provides also something like `X-Appengine-Citylatlong: 51.256213,7.150764`
this could be used deny access from certain regions, but we didn't follow
up on that.

Having country blocking in place an seeing the attack ebbing down we
startet removing the large blocks from `ddos.txt`.


Long term solution - rate limiting
----------------------------------

We didn't want to engage in a arms race with the attackers. We could
start blocking on user agent strings ans such but the best long term
solution would be just to avoid single IP adresses to request too much
data. This to a certain degree also has it's problems, e.g. when many users
come from the same IP due to NAT or proxies.

We where ready to implement such a scheme along the lines of
[Simon Wilson's Blogpost](http://blog.simonwillison.net/post/57956846132/ratelimitcache)
about using memcache buckets for rate limiting but then the DDoS attacks
stopped and we fond more interesting things to do.
