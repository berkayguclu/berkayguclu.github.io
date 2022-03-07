---
layout: post
title:  "TryHackMe/Oh My WebServer"
author: "Berkay Guclu"
lang: tr
categories: tryhackme medium
permalink: /:categories/oh-my-webserver
---

[<img src="/assets/images/tryhackme/webserver.png" height="199">](https://tryhackme.com/room/ohmyweb)

*"Can you root me?"* -[tinyb0y](https://tryhackme.com/p/tinyb0y)

Difficulty: *Medium*

1. [Scan/Enumeration](#scan/enumeration)
2. [Gain Shell](#gain-shell)
3. [Privilege Escalation](#privilege-escalation)


## [Scan/Enumeration]

Makinedeki açık portları tespit edebilmek için, öncelikle bütün portları tarayan bir nmap taraması çalıştırıyorum daha sonra bu tarama sonucunda bulduğum portları daha ayrıntılı incelemek için nmap'in `-A` parametresiyle agresif tarama gerçekleştiriyorum.

{% highlight bash %}
nmap -T4 -vv -p- <ip>
nmap -A -p 22,80 <ip> -oN web.nmap
{% endhighlight %}

Tarama sonucunda makinede 22 portunda OpenSSH servisinin, 80 portunda ise Apache web server'ın çalıştığını görüyorum.

![webserver1](/assets/images/tryhackme/webserver1.png)

Website üzerinde keşif yapabilmek için gobuster yardımıyla dizin taraması yapıyorum. `/assets` ve `/cgi-bin` dışında önemli bir dizin bulamıyorum. Assets dizinine baktığımda dizinde .DS_Store dosyasıyla karşılaşıyorum. Bana bir faydası dokunacak mı bilmesem de bu dosya MacOS işletim sisteminin oluşturduğu bir dosya olduğu için websitesi oluşturulurken mac cihaz kullanıldığını anlıyorum. Dizin tararken bulduğum `/cgi-bin` dizinini istismar edebilmek için apache'nin kullanılan versiyonunda herhangi bir zafiyet var mı kontrolünü yapıyorum. Apache'nin kullanılan versiyonunda bir [Path Traversal & RCE](https://www.exploit-db.com/exploits/50383) zafiyeti olduğunu görüyorum. Sitenin /cgi-bin dizininden Path Traversal veya diğer adıyla LFI zafiyeti aracılığıyla istediğimiz dizine ulaşabiliyoruz. İstediğimiz dizine ulaştığımız HTTP request'inin header'larına istediğimiz komutu yazarak ulaştığımız dosyaya bu komutu input olarak verebiliyoruz. Dolayısıyla LFI ile /bin/sh dosyasına ulaşıp, request'in header'larından da istediğimiz komutları göndererek server'da uzaktan kod çalıştırma (RCE) zafiyetini sömürebiliyoruz.

{% highlight bash %}
curl 'http://<IP>/cgi-bin/.%%32%65/.%%32%65/.%%32%65/.%%32%65/.%%32%65/.%%32%65/bin/sh' --data 'echo Content-Type: text/plain; echo; <command>'
{% endhighlight %}

![webserver2](/assets/images/tryhackme/webserver2.png)


## [Gain Shell]

Server'dan reverse shell alabilmek için payload'ı direkt olarak header kısmına yerleştirmeyi deniyorum fakat bu şekilde shell alamıyorum. Server üzerinde /tmp dizininde bir binary oluşturup paylaod'ı buraya yazdıktan sonra o dosyayı çalıştırarak shell almayı deniyorum ve başarıyorum.

![webserver3](/assets/images/tryhackme/webserver3.png)

`/etc/passwd` dosyasını kontrol ettiğimde makinede yalnızca root kullanıcısının bulunduğunu görüyorum. `/` dizinine baktığımda ise `.dockerenv` dosyasıyla karşılaşıp docker içerisinde olduğumu anlıyorum.
Docker içerisinde de olsam root kullanıcısına yükselebilmek için SUID dosyalarını kontrol ediyorum fakat kullanabileceğim bir binary göremiyorum. Cron job'ları kontrol etmeye çalışıyorum fakat herhangi bir şey bulamıyorum. Capabiliti'leri kontrol ettiğimde ise python'ın ayarlandığını görüyorum. [GTFOBins](https://gtfobins.github.io/gtfobins/python/#capabilities)'in yardımıyla bunu sömürerek docker içerisindeki root kullanıcısına erişiyorum.


![webserver4](/assets/images/tryhackme/webserver4.png)


## [Privilege Escalation]

`ifconfig` komutu ile bulunduğum ip bloğununu kontrol ediyorum ve docker makinemizin `172.17.0.2` ip'sine sahip olduğunu görüyorum. Dolayısıyla üzerinde çalıştığımız makinenin aynı ip bloğundaki router cihazının ip adresine sahip olması gerektiğini düşünerek `172.17.0.1` adresine nmap taraması yapmayı deniyorum. Bulundugum docker'dan ana makineyi keşfedebilmek için makineye bir nmap binary'si atıyorum ve taramayı gerçekleştiriyorum.

![webserver5](/assets/images/tryhackme/webserver5.png)

Makinede karşılaştığımız 5985 ve 5986 portlarında hangi servislerin çalışıyor olabileceğine dair araştırma yapıyorum. Öncelikle WinRM (Windows Remote Management) servisinin olabileceğine dair bilgilerle karşılaşıyorum fakat docker'da çalışan makinenin linux olmasından dolayı ana makinenin de linux olması gerektiğini düşünerek bu servisin olamayacağına kanaat getiriyorum. Yine bu portlar üzerinde çalışan OMI isimli yine microsoft'un geliştirdiği başka bir remote configuration management tool'u ile karşılaşıyorum. Çalışan servisin bu olabileceğini düşünüp bu servise ait exploit aramaya başlıyorum ve bir [github reposu](https://github.com/horizon3ai/CVE-2021-38647) ile karşılaşıyorum. Burada bulduğum exploit'i makineye atıp çalıştırmayı deniyorum ve ana makinede root kullanıcısı olarak komut çalıştırabildiğimi görüyorum.

![webserver6](/assets/images/tryhackme/webserver6.png)

Bu sefer de direkt olarak -c parametresi ile reverse shell almaya çalışıyorum fakat başaramıyorum. Docker'da kullandığım gibi /tmp dizininde bir dosya oluşturarak ana makinede root kullanıcısı olarak shell almayı deniyorum fakat komutlarla direkt olarak dosya oluşturup içine yazamıyorum. Dosyayı kendi makinemde oluşturup ana makineye `wget` aracılığıyla çekip çalıştırarak shell alabiliyorum.

![webserver7](/assets/images/tryhackme/webserver7.png)
