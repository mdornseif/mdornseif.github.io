---
layout: post
title: Generating and Printing PDFs on Google AppEngine (en)
---

{{ page.title }}
================

How we print PDFs on AppEngine (and how we came to use GAE)

I like to share our story on how we handle PDF generation and Printing on GAE. Perhaps some background first (skip the next paragraphs if you are purely interested in PDF generation): 

We are a german sporting goods manufacturer. Our wholesale division is processing and shipping a few hundred
thousand orderlines per year ranging from a small package up to truckloads of the same product. We run a
rather diverse infrastructure starting with IBM Midrange Systems (AS/400) and - till 2010 - ending with
Amazon and Rackspace Cloud offerings. Most newer Software we have developed is based on Django. We have been aware of Google AppEngine but ignored it, because it was "beta" and seemed to see not much comittment by Google. The anouncement of GAE for Buiseness changed that - although the pricing model is not very attractive to us. But ist shows that Google Inc. is willing to see GAE Developers as their customer and not their Product.

Generally we have been looking for PAAS offerings. We are able to build much more complex things than we are
able to monitor and operate. Beeing a relatively small team of 6 people, it would use up a disproportional
amount of our resources to build a automated deployment setup and operate it. We really embraced Ian
Bicking's [Silverlining][1] but after using it for some monthes we had to conclude it wasn't providing us
with the reduction in administration and deployment we were hoping for.

Revisiting GAE we found out that it wsa providing most of what we needed except that it generally was considered slow, unreliable and the support sub-standart. Nobody to call if thinks break.  We consider Cron Jobs, Task Queues, and XMPP integration as killer features: In our experience it is qoute hard to handle installation and monitoring of cronjobs in a cloud infrastructure in a reasonable manner. While there are Things like [Celery][2] operating it in a redundant robust manenr is a lot of hassle to us. It's easy to set up, but monitoring. capacy planning, fail-over, testing, updating under load etc. I really wonder how other small teams manage that. To us it was a burden.

Interestingly I never considered most limitations of GAE important. Coming from a "dependable sistems" CS background I always was very aware of the fact that most operations on a computer can fail and your code should be prepared to handle that - at lest unles you are willing to pay for a [Tandem/NonStop][3] installation. I'm the kind of person who checks the return code not only on malloc() but also on write() and close(). Coming from that mindset it is comforting when there is a "the datastore is down" exception. I find it much more comforting then the "just assume the SQL Server is always up".

Thus in practice it turned out that the errors we see in GAE are at point's we didn't expect. E.g. it is highly annoying and happen's to often that we cant deploy new versions because the deployment infrastructure throws 500 errors. But all in all we are happy GAE is there. 


[1]: http://cloudsilverlining.org/ 
[2]: http://celeryproject.org/
[3]: http://h20223.www2.hp.com/nonstopcomputing/cache/76385-0-0-0-121.html?404m=cache-aspx


PDF Generation
--------------

We need to print packing lists, delivery notes and invoices - a few ten thousand pages a month. The standard way for PDF generation on python seems to be the [ReportLab Toolkit][4] but the open source variant always seemt to low-level for us and for the commercial variant ... ley's say an an engeneer Im not willing to accept per page licensing for a pice of software and the usage restrictions are redicilous. We also wanted a graphical design tool.

A few years ago we decided that [JasperReports][5] together with the [iReport][6] Design Toolis the only viable alternative. Originaly we used a fork(3) from Python of the Java VM but then moved to a [Jython][7] based Server which was accessed via the [PyRO][8] RPC toolkit. Later we moved communication to HTTP and published the whole thing unter the name [pyJasper][9]. The JasperReports can not be Used on GAE/Java, because the used libraries are missing form GAE/Java which is a shame.

The pyJasper (Python) client sends an XML based "Report Description" and the Data to be rendered on the
report via a HTTP POST request to the (Jython/Java) Server, the server renders the PDF optionally signes it
and delivers it back to the client. This works nicely with appengine although the generation and signing of
complex PDFs often takes more than 10s - which was until the 1.4.1 GAE Update a huge issue. We solved that by implementing aggressive caching of generated PDFs in the server.

Now if a request to generate a PDF times out on the client (GAE) side the server still finishes PDF generation and caches the result. The Client using task ques wil shortly retry to generate the PDF and this time we hit the cache and can serve the reply well under 10 seconds.

We currently run the two pyJasper instances on Amazon EC2 behind an "Elastic Load Balancer". This model served us very well and is very simple to operate. We use [`huTools.pyjasper`][9a] to access the Service.

[4]: http://www.reportlab.com/software/opensource/
[5]: http://jasperforge.org/project/jasperreports
[6]: http://jasperforge.org/projects/ireport
[7]: http://www.jython.org/
[8]: http://www.xs4all.nl/~irmen/pyro3/
[9]: https://github.com/hudora/pyJasper
[9a]: https://github.com/hudora/huTools/blob/master/huTools/pyjasper.py


Printing
--------

The next and harder issue is how to "print from a webapplication" - which seems to be a troublesome issue fot a lot of developers. We are talking about thousands of documents per day so obviously we need a fully automated solution. We also talking about a diverse set of warehouses, dynamic IPs and often very unsophisticated internet access so "tunnel IPP on a high port to a local CUPS server" is a valid but unhelpful answer.

At the point in time we had to solve this there was a lot of talk about Google [Cloud Print][10], Apple [Air
Print][11] and HP [ePrint][12] but the first two where not ready for use and the third seemd to be only
implemented in consumer grade printers. Also a big issue for us was QOS. At least for Coud Print and ePrint Delivery Speed ("time to print") is determined by the infrastructure providers and we can't accept tatencies higher than half a minute. Nobody was willing to guarantee that.

[10]: http://blog.chromium.org/2010/04/new-approach-to-printing.html
[11]: http://www.apple.com/ipad/features/airprint.html 
[12]: http://h30495.www3.hp.com/about/eprint

So it seemed we had to pull from the client side. It would be preferrable th have a "Print Server" Application from a Web Page but "Silent Printing" from within a Webpage is considered a security problem (for good reasons) and it is very hard to get arround that (see [stackoverflow][13] for an overview).

After spending o much time in hunting for a "modern" solution we decided to build native clients. We specified a simple HTTP-based protocol named [Frugal Message Trasfer Protocol (FMTP)][14] and creaded a simple [MS Windows GUI client][15], enabling warehouse staff to choose the printer, give an FMTP-Server url to poll and then run unattended. There is also a much simpler [commandline client][15].

[13]: http://stackoverflow.com/questions/21908/silent-printing-in-a-web-application
[14]: https://github.com/cklein/FMTP
[15]: https://github.com/hudora/FMTP/tree/master/printclient
[16]: https://github.com/hudora/FMTP/blob/master/pull_client/recv_fmtp.py


That's basically the whole setup. Call out th EC2/Java to generte PDFs. Queue them and have a specialized client pull for them anf print them. Works like a charm.