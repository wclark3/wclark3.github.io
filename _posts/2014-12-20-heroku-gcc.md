---
layout: post
title: Installing the GNU Compiler Collection from Source on a Heroku Dyno
---

<p class="mdl-typography--headline">{{ page.title }}</p>

This is Part 1 of a set of posts on how to hack together a buildpack for Heroku that runs <a href="https://www.gnu.org/software/octave/">GNU Octave</a>, the open-source Matlab clone that can solve and simulate medium-scale dynamic macroeconomic models. I'm starting with a Heroku app that runs <a href="http://nodejs.org/">node.js</a>, and all the terminal commands I list here are run in a one-off web dyno that I accessed via <span class="tt">heroku run bash</span>.

<p class="mdl-typography--subhead">Prerequisites</p>

The GNU Compiler Collection (GCC) actually comes loaded on a Heroku web dyno out of the box, but as far as I can tell it comes without a Fortran compiler (which is required to build Octave). The full list of prerequisites can be found <a href="https://gcc.gnu.org/install/prerequisites.html">here</a>, but the three that we'll need to install are
<ul><li><a href="https://gmplib.org/">GNU Multiple Precision Arithmetic Library (GMP)</a></li>
<li><a href="http://www.mpfr.org/">GNU Multiple Precision Floating-Point Reliably (MPFR)</a></li>
<li><a href="</a>">GNU Multiple Precision Complex Library (MPC)</a></li>
</ul>

These are actually pretty easy. They can all be downloaded from the <a href="ftp://ftp.gnu.org/gnu/">GNU FTP archive</a>, and they all have straightforward installation procedures. 

The order in which these packages are installed does matter. GMP can be installed by itself, but MPFR must be compiled with GMP, and MPC must be compiled with GMP and MPFR.

<p class="mdl-typography--subhead">GMP</p>

Download the GMP tarball from the GNU archive and unzip it into a build folder. Make a directory in /app/.heroku/ where the installed files will live. Tell the configure script where the install directory is, then configure, make, and install.

{% highlight console %}
~/.heroku $ curl ftp://ftp.gnu.org/gnu/gmp/gmp-5.1.3.tar.bz2 -s -O
~/.heroku $ tar -xjf gmp-5.1.3.tar.bz2
~/.heroku $ mkdir gmp
~/.heroku $ cd gmp-5.1.3
~/.heroku/gmp-5.1.3 $ ./configure --prefix=/app/.heroku/gmp
~/.heroku/gmp-5.1.3 $ make && make install
{% endhighlight %}

Lastly, we need to add the <span class="tt">lib</span> directory to the <span class="tt">LD_LIBRARY_PATH</span>. The easiest way to do this is navigate to the <span class="tt">lib</span> directory and add that directly to the <span class="tt">LD_LIBRARY_PATH</span> using <span class="tt">$PWD</span>.

{% highlight bash %}
~/.heroku/gmp-5.1.3 $ cd ../gmp/lib
~/.heroku/gmp/lib $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$PWD
{% endhighlight %}

<p class="mdl-typography--subhead">MPFR</p>

We can install MPFR with the same procedure that we used for GMP. The only difference is that we need to point the configure script to the GMP install directory.

{% highlight bash %}
~/.heroku $ curl ftp://ftp.gnu.org/gnu/mpfr/mpfr-3.1.2.tar.xz -s -O
~/.heroku $ tar -xf mpfr-3.1.2.tar.xz
~/.heroku $ mkdir mpfr
~/.heroku $ cd mpfr-3.1.2
~/.heroku/mpfr-3.1.2 $ ./configure --prefix=/app/.heroku/mpfr --with-gmp=/app/.heroku/gmp
~/.heroku/mpfr-3.1.2 $ make && make install
{% endhighlight %}

Once again, add the <span class="tt">lib</span> directory to the <span class="tt">LD_LIBRARY_PATH</span>

{% highlight bash %}
~/.heroku/mpfr-3.1.2 $ cd ../mpfr/lib
~/.heroku/mpfr/lib $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$PWD
{% endhighlight %}

<p class="mdl-typography--subhead">MPC</p>

Repeat for MPC. We need to tell the configure script where GMP and MPFR are installed.

{% highlight bash %}
~/.heroku $ curl ftp://ftp.gnu.org/gnu/mpc/mpc-1.0.2.tar.gz -s -O
~/.heroku $ tar -xzf mpc-1.0.2.tar.gz
~/.heroku $ mkdir mpc
~/.heroku $ cd mpc-1.0.2
~/.heroku/mpc-1.0.2 $ ./configure --prefix=/app/.heroku/mpc --with-gmp=/app/.heroku/gmp --with-mpfr=/app/.heroku/mpfr
~/.heroku/mpc-1.0.2 $ make && make install
{% endhighlight %}

Again, add the <span class="tt">lib</span> directory to the <span class="tt">LD_LIBRARY_PATH</span>

{% highlight bash %}
~/.heroku/mpc-1.0.2 $ cd ../mpc/lib
~/.heroku/mpc/lib $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$PWD
{% endhighlight %}

<p class="mdl-typography--subhead">Installing GCC</p>

Now we're ready to install GCC. It follows more or less the same procedure. Normally, GCC will look for GMP, MPC, and MPFR in <span class="tt">/usr/bin/</span> or some similar directory, but since we have these installed in a non-standard location, we will have to tell GCC's configure script where they are. We can do this by passing arguments to the configure script that list the install directories we used in the previous steps.

The option <span class="tt">--disable-multilib</span> is oddly named, but it tells the configure script that we only want a 64-bit compiler (not a 32-bit compiler).

Running the configure script will take a few minutes, but running the make script will take a few hours (around 4, I think), so get it going and then go get a cup of coffee or read a book or something.

{% highlight bash %}
~/.heroku $ curl ftp://ftp.gnu.org/gnu/gcc/gcc-4.9.2/gcc-4.9.2.tar.gz -s -O
~/.heroku $ tar -xzf gcc-4.9.2.tar.gz
~/.heroku $ mkdir gcc
~/.heroku $ cd gcc-4.9.2
~/.heroku/gcc-4.9.2 $ ./configure --prefix=/app/.heroku/gcc --with-gmp=/app/.heroku/gmp/ --with-mpfr=/app/.heroku/mpfr/ --with-mpc=/app/.heroku/mpc/ --disable-multilib
~/.heroku/gcc-4.9.2 $ make && make install
{% endhighlight %}

This time, we need to add the <span class="tt">lib64</span> directory to the <span class="tt">LD_LIBRARY_PATH</span>

{% highlight bash %}
~/.heroku/gcc-4.9.2 $ cd ../gcc/lib64
~/.heroku/gcc/lib64 $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$PWD
{% endhighlight %}

Now GCC is ready for us to use to <a href="{% post_url 2014-12-27-heroku-octave %}">install Octave from source</a>.