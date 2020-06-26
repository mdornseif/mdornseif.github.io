---
layout: post
title: Authentication iPhones to AppEngine (en)
---

{{ page.title }}
================

You want to be sure the data sent from the Client to your Webserver is genuide.
This is a suprisingly hard problem. And unfortunately "just use SSL/TLS" does
not solve that problem.

For this article we assume the following

* the client device (an iPhone) and the server have already agreed on a shared secret
* we need to ensure the integrety and authenticy of the data sent by the client via POST and PUT requests
* we assume SSL/TLS can provide a reasonable amount of confidentiality

The simplest aproach would be to just include a secret token the data send by the client
and hope SSL/TLS would ensure the secret token stays secret. Unfortunately on AppEngine
we have very little control over the SSL/TLS process, so we can't use fancy things like
client certificats or harden the SSL/TLS configuration. Also we ususally have to share
a Vertivicat with thousand of other apps in the *.appspot.com` domain.
SSL/TLS can be attacked on the wire by man in the middle attacks and especially
if they are done by the device owner.

Such an attack would completely destroy confidentiality, integry checks and authenticitation

Algorythm:

* hash the message Body
* hmac the HTTP-Method, the HTTP-Path and the hash of the message body:
* put that into the authorisation header.
* hashes are base64 encoded
* fields should be ASCII only but in case they are not utf-8 encoded and `&` separated.

So the Server site looks like this:

<script src="https://gist.github.com/mdornseif/6816959.js"></script>

The client site is somewhat longer due to Objective-C`s wordieness. We subclass AFNetwork. This example only implements the PUT Method but you get an idea.
