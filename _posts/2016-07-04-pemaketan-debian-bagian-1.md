---
layout: post
title: "Pemaketan Debian - Bagian 1"
description: "Pedoman pemaketan dasar Debian."
modified: 2016-08-15
tags: [debian, blankon, pemaketan]
comments: true
---

## Pedoman pemaketan dasar Debian.

 Pada tulisan ini kita akan belajar membuat paket untuk Debian dan distro turunannya, kita akan mencoba beragam cara membuat paket Debian, saya memisahkan Pedoman ini dalam beberapa bagian dan berusaha agar pembaca dapat dengan mudah memahaminya.

**Daftar Isi**

* TOC
{:toc}
 
### Persiapan.

#### Pemasangan paket yang diperlukan.

{% highlight bash %}
$ sudo apt-get install -y devscripts build-essential fakeroot debhelper \
gnupg pbuilder dh-make dpkg-dev ubuntu-dev-tools \
autotools-dev lintian rng-tools
{% endhighlight %}

<div class="alert alert-note"><strong>Catatan:</strong>
<p>Aplikasi <em>rng-tools</em> di gunakan disini untuk mempercepat proses pembuatan kunci GPG.</p> 
</div>

#### Konfigurasi awal informasi tentang pemaket.

Mengatur variabel $DEBFULLNAME dan $DEBEMAIL pada lingkungan shell sehingga berbagai alat-alat pemaketan Debian mengenali **Nama** dan **Alamat Email** Anda untuk memaketkan paket.

{% highlight bash %}
$ echo 'export DEBFULLNAME="Nama Anda"' >> ~/.bashrc
$ echo 'export DEBEMAIL="email@anda.com"' >> ~/.bashrc

$ source ~/.bashrc
{% endhighlight %}

 atau

{% highlight bash %}
$ echo 'export DEBFULLNAME="Nama Anda"' >> ~/.profile
$ echo 'export DEBEMAIL="email@anda.com"' >> ~/.profile

$ source ~/.profile
{% endhighlight %}

 atau	

{% highlight bash %}
$ . ~/.bashrc
{% endhighlight %}

 atau

{% highlight bash %}
$ . ~/.profile
{% endhighlight %}

atau bisa juga seperti ini:

{% highlight bash %}
$ cat >> ~/.bashrc <<EOF
DEBFULLNAME="Nama Anda"
DEBEMAIL="email@anda.com"
export DEBFULLNAME DEBEMAIL
EOF
$ . ~/.bashrc
{% endhighlight %}

 Periksa hasilnya

{% highlight bash %}
$ echo $DEBFULLNAME ; echo $DEBEMAIL
$ export | grep DEB
$ grep DEB* ~/.bashrc
$ grep DEB* ~/.profile
{% endhighlight %}

 Hasilnya akan menampilkan informasi tentang pemaket, pilih salah satu antara merubah berkas 
 *.bashrc* atau *.profile* karena Banyak jalan menuju Roma :)
 
#### Membuat kunci GnuPG (GnuPrivacyGuard).

 Sebelum membuat kunci GPG, kita jalankan daemon `rngd` secara manual:

{% highlight bash %}
$ sudo rngd -r /dev/urandom
{% endhighlight %}

 Kita juga perlu memperbaharui GnuPG untuk menggunakan **SHA 2** mengacu pada **SHA1**.
 Jadi tambahkan pada akhir file `~/.gnupg/gpg.conf`:

{% highlight bash %}
personal-digest-preferences SHA256
cert-digest-algo SHA256
default-preference-list SHA512 SHA384 SHA256 SHA224 AES256 AES192 AES CAST5 ZLIB BZIP2 ZIP Uncompressed
{% endhighlight %}

 Berikut ini detailnya:
 
{% highlight bash %}
$ mkdir -p ~/.gnupg/
$ cat >> ~/.gnupg/gpg.conf <<EOF
personal-digest-preferences SHA256
cert-digest-algo SHA256
default-preference-list SHA512 SHA384 SHA256 SHA224 AES256 AES192 AES CAST5 ZLIB BZIP2 ZIP Uncompressed
EOF
{% endhighlight %}

 Lanjutkan dengan proses membuat kunci GPG:

{% highlight bash %}
$ gpg --gen-key
{% endhighlight %}		

 Pilih: *(1) RSA and RSA (default) -> 4096 -> (0) -> y -> (O)kay*
 
 Untuk melihat hasil dari pembuatan kunci GPG, bisa dilakukan dengan perintah berikut:
 
{% highlight bash %}
$ gpg --list-key --fingerprint
{% endhighlight %}
 
 Setelah proses pembuatan kunci GPG selesai, hentikan daemon `rngd` dengan perintah `pkill`:
 
{% highlight bash %}
$ sudo pkill rngd
{% endhighlight %}

 Pastikan daemon `rngd` tidak berjalan:

{% highlight bash %}
$ ps ax | grep rngd
15459 pts/1    S+     0:00 grep rngd
{% endhighlight %}

 Hapus paket `rng-tools` bila tidak ingin membuat kunci gpg di kemudian hari.
 
{% highlight bash %}
$ sudo apt-get remove rng-tools
{% endhighlight %}
 
#### Membuat direktori kerja.

 Sebelum kita memulai pada tahap selanjutnya, mari kita membuat direktori kerja kemudian masuk ke direktori
 yang telah kita buat:

{% highlight bash %}
$ mkdir -p ~/Belajar/Pemaketan/{src,final,github}
$ cd ~/Belajar/Pemaketan/src
{% endhighlight %}

 Hasil dari membuat direktori kerja akan tampak seperti ini:
 
{% highlight bash %}
$ tree ~/Belajar
/home/user-anda/Belajar
└── Pemaketan
    ├── final
    ├── github
    └── src

4 directories, 0 files
{% endhighlight %}

<div class="alert alert-note"><strong>Keterangan:</strong>
<ul>
<li>Direktori <em>src</em> digunakan untuk membangun paket Debian.</li>
<li>Direktori <em>final</em> digunakan untuk menyimpan paket yang akan kita buat.</li>
<li>Direktori <em>github</em> digunakan untuk membuat repositori Git dari paket yang akan kita buat.</li>
</ul>
</div>

#### Unduh paket sumber.

 Anda bisa memilih paket apa yang mau di buatkan versi Debiannya, disini saya 
 memilih paket **[GNU ed](http://ftp.gnu.org/gnu/ed/)**. Saat artikel ini ditulis 
 versi terakhir adalah 1.9.
 
 Mulai mengunduh paket sumber

{% highlight bash %}
$ wget http://ftp.gnu.org/gnu/ed/ed-1.9.tar.gz
{% endhighlight %}

 Ekstrak paket sumber dan masuk ke direktori hasil ekstrak

{% highlight bash %}
$ tar zxvf ed-1.9.tar.gz
$ cd ed-1.9
{% endhighlight %}

 Kita dapat melihat struktur direktorinya dengan perintah *tree*

{% highlight bash %}
$ tree -d ../
../
`-- ed-1.9
	|-- doc
	`-- testsuite
{% endhighlight %}

 Bila perintah tidak ditemukan silahkan pasang paketnya

{% highlight bash %}
$ sudo apt-get install -y tree
{% endhighlight %}

### Memulai membangun paket.

 Mari kita mulai membangun paket yang telah kita pilih sebelumnya

{% highlight bash %}
$ dh_make -e $DEBEMAIL -c gpl3 -s -f ../ed-1.9.tar.gz
{% endhighlight %}

 Kita bisa melihat perubahan yang terjadi pada struktur direktorinya

{% highlight bash %}
$ tree -d -D -t ../
../
`-- [Jul  3  3:41]  ed-1.9
	|-- [Jul  3  3:38]  doc
	|-- [Jul  3  3:38]  testsuite
	`-- [Jul  3  3:41]  debian
		`-- [Jul  3  3:41]  source


$ tree -D -t debian/
debian/
|-- [Jul  3  3:41]  README.Debian
|-- [Jul  3  3:41]  README.source
|-- [Jul  3  3:41]  changelog
|-- [Jul  3  3:41]  compat
|-- [Jul  3  3:41]  control
|-- [Jul  3  3:41]  copyright
|-- [Jul  3  3:41]  docs
|-- [Jul  3  3:41]  ed.cron.d.ex
|-- [Jul  3  3:41]  ed.default.ex
|-- [Jul  3  3:41]  ed.doc-base.EX
|-- [Jul  3  3:41]  info
|-- [Jul  3  3:41]  init.d.ex
|-- [Jul  3  3:41]  manpage.1.ex
|-- [Jul  3  3:41]  manpage.sgml.ex
|-- [Jul  3  3:41]  manpage.xml.ex
|-- [Jul  3  3:41]  menu.ex
|-- [Jul  3  3:41]  postinst.ex
|-- [Jul  3  3:41]  postrm.ex
|-- [Jul  3  3:41]  preinst.ex
|-- [Jul  3  3:41]  prerm.ex
|-- [Jul  3  3:41]  rules
|-- [Jul  3  3:41]  source
|   `-- [Jul  3  3:41]  format
`-- [Jul  3  3:41]  watch.ex


$ ls ../
ed-1.9  ed-1.9.tar.gz  ed_1.9.orig.tar.gz
{% endhighlight %}

 Dari hasil perintah *tree* kita bisa melihat ada penambahan direktori *debian* dan *debian/source* 
 dan juga salinan dari paket sumber yaitu *ed_1.9.orig.tar.gz*.

 Mari kita hapus beberapa berkas yang saat tidak diperlukan[^1] pada direktori *debian*.

{% highlight bash %}
$ cd debian
$ rm *.ex *.EX README.* *.docs info
{% endhighlight %}

 Hasilnya akan seperti ini

{% highlight bash %}
$ tree -D ../debian/
../debian/
|-- [Jul  3  3:41]  changelog
|-- [Jul  3  3:41]  compat
|-- [Jul  3  3:41]  control
|-- [Jul  3  3:41]  copyright
|-- [Jul  3  3:41]  rules
`-- [Jul  3  3:41]  source
	`-- [Jul  3  3:41]  format
{% endhighlight %}

#### Menyunting berkas changelog, control dan copyright.

 Pada tahap ini kita akan merubah beberapa berkas yang ada di direktori *debian* diawali 
 dengan berkas *changelog*, sebagai catatan...berkas-berkas ini disunting sesuai dengan 
 ketentuan pemaketan Debian, sehingga kita tidak bisa seenaknya menyunting berkas yang ada.
 
##### Sunting berkas changelog.

 Mari kita mulai menyunting berkas *changelog* ala Debian :)

{% highlight bash %}
$ dch -e
{% endhighlight %}

<div class="alert alert-note"><strong>Keterangan:</strong>
<ol>
<li>Perintah <em>dch -e</em> digunakan untuk penyuntingan awal.</li>
<li>Perintah <em>dch -i</em> digunakan untuk penyuntingan selanjutnya ataupun pemaket lainnya.</li>
</ol>
</div>

{% highlight bash %}
ed (1.9-1) unstable; urgency=low

  * Initial release (Closes: #nnnn)  <nnnn is the bug number of your ITP>

 -- Nama Anda <email@anda.com>  Sun, 03 Jul 2016 03:41:14 +0700
{% endhighlight %}

<div class="alert alert-note"><strong>Keterangan:</strong>
<ul>
<li>1.9 merupakan versi upstream/hulu.</li>
<li>1 merupakan versi Debian.</li>
<li><a href="https://www.debian.org/doc/debian-policy/ch-controlfields.html#s-f-Distribution" target="_blank">unstable</a> merupakan kode rilis paket Debian, nilai unstable umumnya digunakan untuk paket baru, versi terbaru dari paket upstream/hulu dan perbaikan kutu, sedangkan nilai experimental umumnya digunakan saat pengembang melakukan uji coba (versi beta) sebelum paket itu dirilis untuk Debian (nilainya experimental, unstable, testing dan stable). Lihat <a href="https://www.debian.org/doc/manuals/developers-reference/ch04.en.html#archive" target="_blank">Debian Developer's Reference</a> dan <a href="https://debian-handbook.info/browse/stable/sect.release-lifecycle.html" target="_blank">The Debian Administrator's Handbook</a> untuk informasi lebih lanjut.</li>
<li><a href="https://www.debian.org/doc/debian-policy/ch-controlfields.html#s-f-Urgency" target="_blank">urgency=low</a> merupakan deskripsi tentang seberapa pentingnya peningkatan versi dari versi sebelumnya (nilainya low, medium, high, emergency, atau critical).</li>
<li><em>nnnn</em> merupakan nomor kutu <a href="https://wiki.debian.org/ITP" target="_blank">Intent to Package (ITP)</a> yang
diperoleh ketika kita melaporkan paket yang ingin ditambahkan/kelola melalui <a href="https://www.debian.org/Bugs/" target="_blank">Debian Bug Tracking System (Debian BTS)</a>.</li>
</ul>
</div>

<div class="alert alert-note"><strong>Catatan:</strong>
<p>Bila ingin memaketkan untuk Debian silahkan hapus keterangan <em>nnnn is the bug number of your ITP</em>.</p> 
</div>

 Menjadi

{% highlight bash %}
ed (1.9-0blankon1) tambora; urgency=low

  * Initial release for Blankon Tambora

 -- Nama Anda <email@anda.com>  Sun, 03 Jul 2016 03:41:14 +0700
{% endhighlight %}

<div class="alert alert-note"><strong>Keterangan:</strong>
<ul>
<li>1.9 merupakan versi upstream/hulu.</li>
<li>0 merupakan versi Debian.</li>
<li>blankon1 merupakan versi Blankon.</li>
<li>tambora merupakan kode rilis Blankon.</li>
</ul>
</div>

##### Sunting berkas control.

 Sunting berkas *control* dengan text editor favorit.

{% highlight bash %}
$ cd ..
$ nano debian/control
		
Source: ed
Section: editors
Priority: optional
Maintainer: Nama Anda <email@anda.com>
Build-Depends: debhelper (>= 9), autotools-dev
Standards-Version: 3.9.9
Homepage: http://www.gnu.org/software/ed/
#Vcs-Git:
Vcs-Browser: http://web.cvs.savannah.gnu.org/viewvc/?root=ed

Package: ed
Architecture: any
Depends: ${shlibs:Depends}, ${misc:Depends}
Description: GNU ed is a line-oriented text editor.
 GNU ed is a line-oriented text editor. It is used to create, display,
 modify and otherwise manipulate text files, both interactively and via
 shell scripts. A restricted version of ed, red, can only edit files in
 the current directory and cannot execute shell commands.
 .
 Ed is the "standard" text editor in the sense that it is the original
 editor for Unix, and thus widely available. For most purposes, however,
 it is superseded by full-screen editors such as GNU Emacs or GNU Moe.
 .
 Extensions to and deviations from the POSIX standard are described below.
{% endhighlight %}

 Pada paket Debian, berkas *debian/control* memiliki [sintaks tertentu](https://www.debian.org/doc/debian-policy/ch-controlfields.html) dimana setiap baris harus dimulai dengan *spasi* dan paragraf dipisahkan oleh sebuah *titik*. Jika ingin mengikuti sintaks [DEP-5](http://dep.debian.net/deps/dep5/) untuk berkas *debian/copyright*, Kita akan memiliki masalah yang sama saat mengisi teks lisensi untuk *creative-common license*: sehingga *spasi* atau *titik* akan terlewatkan.

 Pada berkas ini saya hanya melakukan penyuntingan pada bagian *Section*, *Homepage*, 
 *Vcs-Git*, *Vcs-Browser* dan *Description*. silahkan lihat informasi paket **GNU ed** 
 pada berkas README, untuk informasi lengkap paket ini bisa merujuk pada websitenya.

##### Sunting berkas copyright.

 Langkah berikutnya kita akan menyunting berkas *copyright*, berikut caranya

{% highlight bash %}
$ nano debian/copyright

Format: http://www.debian.org/doc/packaging-manuals/copyright-format/1.0/
Upstream-Name: ed
Source: http://www.gnu.org/software/ed/

Files: *
Copyright: 2006, 2007, 2008, 2009, 2010, 2011, 2012, 2013, Free Software Foundation, Inc.
	   2006, 2007, 2008, 2009, 2010, 2012, 2013, Antonio Diaz Diaz <ant_diaz@teleline.es>
	   1993, Francois Pinard <pinard@icule>
	   1993-1994, Andrew Moore <alm@woops.talke.org>
License: GPL-3.0+

Files: debian/*
Copyright: 2016 Nama Anda <email@anda.com>
License: GPL-3.0+

License: GPL-3.0+
 This program is free software: you can redistribute it and/or modify
 it under the terms of the GNU General Public License as published by
 the Free Software Foundation, either version 3 of the License, or
 (at your option) any later version.
 .
 This package is distributed in the hope that it will be useful,
 but WITHOUT ANY WARRANTY; without even the implied warranty of
 MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 GNU General Public License for more details.
 .
 You should have received a copy of the GNU General Public License
 along with this program. If not, see <http://www.gnu.org/licenses/>.
 .
 On Debian systems, the complete text of the GNU General
 Public License version 3 can be found in "/usr/share/common-licenses/GPL-3".
{% endhighlight %}

 Informasi lisensi bisa dilihat di berkas *AUTHORS* dan *ChangeLog*. Saya hanya 
 menyunting bagian *Upstream-Name* dan *Copyright*.

 Lakukan pengecekan dengan perintah *cme*, bila tidak tersedia silahkan pasang 
 terlebih dahulu

{% highlight bash %}
$ sudo apt-get install -y install cme libconfig-model-dpkg-perl libconfig-model-tkui-perl
$ cme check dpkg
loading data
checking data
check done
{% endhighlight %}

 Apabila hasilnya tidak seperti diatas maka lakukan penyuntingan kembali sesuai 
 dengan output dari perintah *cme*. Anda juga bisa menyunting paket Debian dengan GUI,
 silahkan jalankan diterminal:
 
{% highlight bash %}
$ cme edit dpkg
{% endhighlight %}

 Akan tampil jendela baru seperti gambar dibawah ini
 
<figure>
	<a href="/images/cme-dpkg-001.png"><img src="/images/cme-dpkg-001.png" alt="Langkah 4" width="740px" /></a>
	<figcaption>cme GUI.</figcaption>
</figure>

 Cara penggunaan silahkan lihat [disini][1].
 
### Paketkan.
 
 Langkah selanjutnya kita akan mulai membuat paket Debian.

{% highlight bash %}
$ dpkg-buildpackage -rfakeroot
{% endhighlight %}

<div class="alert alert-warning"><strong>Catatan:</strong>
<p>Tunggulah hingga proses selesai, karena nantinya akan diminta <em>passphrase</em> yang dibuat saat membuat kunci GnuPG. Apabila hal ini terlewati maka pembuatan paket akan galat/gagal.</p></div>

#### Paket versi BlankOn.

 Lihat hasilnya

{% highlight bash %}
$ ls ../ | grep ed
ed-1.9
ed-1.9.tar.gz
ed_1.9-0blankon1.debian.tar.xz
ed_1.9-0blankon1.dsc
ed_1.9-0blankon1_armhf.changes
ed_1.9-0blankon1_armhf.deb
ed_1.9.orig.tar.gz
{% endhighlight %}

#### Paket versi Debian.

 Bila memaketkan ke versi Debian hasilnya seperti ini
 
{% highlight bash %}
$ ls ../ | grep ed
ed-1.9
ed-1.9.tar.gz
ed_1.9-1.debian.tar.xz
ed_1.9-1.dsc
ed_1.9-1_armhf.changes
ed_1.9-1_armhf.deb
ed_1.9.orig.tar.gz
{% endhighlight %}

 Saya membangun paket pada mesin [ARM Raspberry 2 model B](https://www.raspberrypi.org/products/raspberry-pi-2-model-b/),
 hasil yang ditampilkan akan berbeda bila Anda membangun paket pada mesin berarsitektur 32-bit dan 64-bit, saya merekomendasikan Anda untuk membangun paket pada mesin 64-bit. Silahkan lihat arsitektur apa saja yang di dukung
 pada halaman [Ports website Debian](https://www.debian.org/ports/#portlist-released).

### Pemeriksaan paket.

 Periksa paket dengan menggunakan *lintian*	

{% highlight bash %}
$ lintian -iIEv --pedantic ../*.changes
{% endhighlight %}

 Selesai...sampai nanti di tulisan berikutnya.

## Rujukan

+ [Debian Policy Manual.](https://www.debian.org/doc/debian-policy/)
+ [Debian New Maintainers' Guide.](https://www.debian.org/doc/manuals/maint-guide/)
+ [Guide for Debian Maintainers.](https://www.debian.org/doc/manuals/debmake-doc/)
+ [Panduan Pembuatan Paket BlankOn.](http://dev.blankonlinux.or.id/wiki/Pemaket/PanduanPembuatanPaketMotu)
+ [Managing Debian packages with cme](https://github.com/dod38fr/config-model/wiki/Managing-Debian-packages-with-cme)
+ [Updating debian copyright file with cme](https://github.com/dod38fr/config-model/wiki/Updating-debian-copyright-file-with-cme)
+ [Creating a new GPG key](https://keyring.debian.org/creating-key.html)

---

[1]: https://github.com/dod38fr/config-model/wiki/Managing-Debian-packages-with-cme
[^1]: Dilain kesempatan kita akan bahas bagaimana cara menyunting berkas *postinst, postrm, preinst, prerm, dll*.
