---
layout: post
title:  "TryHackMe/Dav"
author: "Berkay Guclu"
lang: en
categories: tryhackme easy
permalink: /:categories/dav
---

[<img src="/assets/images/tryhackme/dav.jpeg" height="199">](https://tryhackme.com/room/bsidesgtdav)

*"boot2root machine for FIT and bsides guatemala CTF"* -[stuxnet](https://tryhackme.com/p/stuxnet)

Difficulty: *Easy*

1. [Scan/Enumeration](#scan/enumeration)
2. [Gain Shell](#gain-shell)
3. [Privilege Escalation](#privilege-escalation)


## [Scan/Enumeration]

We can start by doing an nmap scan.

{% highlight bash %}
nmap -sVC -oN dav.nmap <IP>
{% endhighlight %}

![dav-1](/assets/images/tryhackme/dav-1.png)

As you can see, only port 80 is open. If you look at the site you will see that it is a normal apache page. We can do directory scan with the gobuster. I will use [common.txt](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/common.txt).

{% highlight bash %}
gobuster dir -u http://<IP>/ -w <wordlist>
{% endhighlight %}

![dav-2](/assets/images/tryhackme/dav-2.png)

We found a directory called "webdav". We need a password to enter this directory. So let's search this: "webdav default password"

![dav-3](/assets/images/tryhackme/dav-3.png)

Try this information and you will see that it works.


## [Gain Shell]

If we are able to install [php reverse shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) on the page we are logged in, we can get a shell. We will use the cadaver tool to do this.

{% highlight bash %}
cadaver http://<IP>/webdav
{% endhighlight %}

![dav-4](/assets/images/tryhackme/dav-4.png)

Now we can upload file with "put" command.

{% highlight bash %}
put php-reverse-shell.php
{% endhighlight %}

Now, while listening to the port with nc, we need to run the file we uploaded through the site.

{% highlight bash %}
nc -nvlp <port>
{% endhighlight %}

![dav-5](/assets/images/tryhackme/dav-5.png)


## [Privilege Escalation]

We can use the "sudo -l" command to see the commands we can use with the sudo authority.

{% highlight bash %}
sudo -l
{% endhighlight %}

![dav-6](/assets/images/tryhackme/dav-6.png)

We can use cat command. We can read the /root/root.txt file.

![dav-7](/assets/images/tryhackme/dav-7.png)
