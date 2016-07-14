---
layout: post
title: "OpenWrt+Tor firmware untuk GL.iNet 6416 dan GL.iNet AR150"
description: "Upgrade GL.iNet firmware dengan OpenWrt-Tor."
tags: [openwrt, glinet, firmware, 6416, ar150]
image:
  background: triangular.png
comments: true
---

## Upgrade GL.iNet firmware dengan OpenWrt-Tor.

 Berhubung semalam dengan ditemani oleh [Mahyuddin Ramli](http://dotovr.blogspot.com/) saya melakukan
 upgrade firmware GL.iNet 6416A, maka saya tuliskan disini agar pembaca bisa meng-upgradenya juga.
 
 Namun perlu di tegaskan lebih dulu...
 
<div class="alert alert-danger"><strong>Hati-hati:</strong>
<p>Kerusakan perangkat yang disebabkan karena mengikuti langkah-langkah ini bukanlah tanggung jawab
saya selaku penulis.</p></div> 

**Daftar Isi**

* TOC
{:toc}
 
### Persiapan.

#### Unduh firmware OpenWrt+Tor.

{% highlight bash %}
$ wget http://www.gl-inet.com/firmware/6416/tor/openwrt-6416-tor-1.3.bin
{% endhighlight %}

 Bila Anda menggunakan GL.iNet versi AR150:
 
{% highlight bash %}
$ wget http://www.gl-inet.com/firmware/ar150/tor/openwrt-ar150-tor-1.3.bin
{% endhighlight %}

<div class="alert alert-note"><strong>Catatan:</strong>
<p>Untuk versi sebelumnya silahkan menuju ke tautan: <a href="http://www.gl-inet.com/firmware/6416/tor/" target="_blank">
Tor 6416</a> dan <a href="http://www.gl-inet.com/firmware/ar150/tor/" target="_blank">Tor AR150</a></p></div>

#### Konfigurasi awal.

 Silahkan lihat video berikut:
 
<iframe width="560" height="315" src="//www.youtube.com/embed/EYBdryKP7fc" frameborder="0"> </iframe>

 Setelah upgrade firmware selesai, Anda bisa mengakses jaringan Tor melalui SSID **tor** dengan
 password **goodlife** atau melalui **kabel LAN**, sedangkan SSID **OpenWrt** dengan password **goodlife** untuk mengakses
 jaringan tanpa Tor sekaligus mengkonfigurasi GL.iNet.
 
 Anda bisa mengganti password serta SSID **tor** dan **OpenWrt** dengan mengakses SSID **OpenWrt**
 terlebih dahulu, berhubung saya menggunakan GL.iNet sebagai repeater untuk terhubung ke jaringan
 **@wifi.id** maka:
 
 + Ubah **INTERNET CONFIGURATION** dari **PROTOCOL DHCP** ke **WiFi**
 melalui [Domino Web Panel](http://192.168.8.1/cgi-bin/luci/webpanel/).
 + Masuk ke [LUCI Web Panel](http://192.168.8.1/cgi-bin/luci/admin/) pilih menu
 *Network > Interfaces* dan ubah *network interface* **LAN**, **TOR** dan **TOR1**
 dengan memilih *Edit*, pada *DHCP Server* pilih tab *Advenced Settings*, isikan kolom
 *DHCP-Options* dengan alamat *DNS* **@wifi.id**,
 contoh: untuk interface **LAN** saya isikan *6,192.168.8.1,10.232.0.4,118.98.44.10,8.8.8.8,8.8.4.4*
 saya tambahkan juga *Google DNS*, untuk pilihan **DNS** terakhir terserah Anda. Kemudian *Save & Apply*.
 + Masih di [LUCI Web Panel](http://192.168.8.1/cgi-bin/luci/admin/) pilih menu
 *Network > Firewall > Custom Rules* dan ubah pada:
 
{% highlight bash %}
enable_transparent_tor() {
  iptables -t nat -A PREROUTING -i wlan0 -p udp --dport 53 -j REDIRECT --to-ports 9053
  iptables -t nat -A PREROUTING -i wlan0 -p tcp --syn -j REDIRECT --to-ports 9040 
  iptables -t nat -A PREROUTING -i eth1 -p udp --dport 53 -j REDIRECT --to-ports 9053
  iptables -t nat -A PREROUTING -i eth1 -p tcp --syn -j REDIRECT --to-ports 9040
}
enable_transparent_tor
{% endhighlight %}

 Menjadi
 
{% highlight bash %}
enable_transparent_tor() {
  iptables -t nat -A PREROUTING -i wlan0-1 -p udp --dport 53 -j REDIRECT --to-ports 9053
  iptables -t nat -A PREROUTING -i wlan0-1 -p tcp --syn -j REDIRECT --to-ports 9040 
  iptables -t nat -A PREROUTING -i eth1 -p udp --dport 53 -j REDIRECT --to-ports 9053
  iptables -t nat -A PREROUTING -i eth1 -p tcp --syn -j REDIRECT --to-ports 9040
}
enable_transparent_tor
{% endhighlight %}

 + Akses GL.iNet melalui ssh dan masukkan password root Anda.

{% highlight bash %}
ssh root@192.168.8.1
{% endhighlight %}

 + Lihat interface untuk SSID **tor**.

 Karena saya memfungsikan GL.iNet sebagai repeater maka interface wlan0 akan bertambah,
 lihat *ip address* yang digunakan pada jaringan **tor**:
 
{% highlight bash %}
 # ifconfig
{% endhighlight %} 

 + Sunting berkas */etc/init/tor*.
 
{% highlight bash %}
 # vim /etc/init.d/tor
{% endhighlight %}
 
 ubah

{% highlight bash %} 
 start() {
        while [ -z "$(ifconfig wlan0)" -a -z "$(ifconfig eth1)" ]; do
                sleep 5
        done
{% endhighlight %}
		
 menjadi

{% highlight bash %} 
 start() {
        while [ -z "$(ifconfig wlan0-1)" -a -z "$(ifconfig eth1)" ]; do
                sleep 5
        done
{% endhighlight %}

 Silahkan jalankan ulang *service firewall*:

{% highlight bash %}
# /etc/init.d/firewall restart
{% endhighlight %}

 Bila sampai tahap ini Anda belum bisa mengakses jaringan internet, hidupkan ulang GL.inet:

{% highlight bash %}
# reboot
{% endhighlight %}
 
 Selesai...silahkan menikmati akses internet secara anonym melalui Tor dengan menghubungkan
 perangkat Anda ke SSID **Tor** atau kabel **LAN** dan sampai nanti di tulisan berikutnya.

## Rujukan

+ [New Tor firmware for GL-AR150 and GL.iNet6416](http://www.gl-inet.com/new-tor-firmware-for-gl-ar150-and-gl-inet6416/)
+ [Using Tor with Repeater mode](http://www.gl-inet.com/using-tor-with-repeater-mode/)