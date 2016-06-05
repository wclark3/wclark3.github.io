---
layout: post
title: Installing GNU Octave from Source on a Heroku Dyno
---

<h1>{{ page.title }}</h1>

This is Part 2 of a set of posts on how to hack together a buildpack for Heroku that runs <a href="https://www.gnu.org/software/octave/">GNU Octave</a>, the open-source Matlab clone that can solve and simulate medium-scale dynamic macroeconomic models. I'm starting with a Heroku app that runs <a href="http://nodejs.org/">node.js</a>, and all the terminal commands I list here are run in a one-off web dyno that I accessed via <span class="tt">heroku run bash</span>.

In <a href="{% post_url 2014-12-20-heroku-gcc %}">Part 1</a>, I discussed how to install the GNU Compiler Collection, and I will assume here that GCC is already installed and that <span class="tt">../gcc/bin</span> has been added to the <span class="tt">PATH</span> environment variable.

<h2>Prerequisites</h2>

The official list of prerequisites can be found <a href="https://www.gnu.org/software/octave/doc/interpreter/External-Packages.html#External-Packages" target="_blank" >here</a>. Some of these are already installed on a Heroku dyno, and some will not be needed since we are not going to use Octave's graphics functionality. We will need to install:
<ul><li><a href="http://www.netlib.org/blas">Basic Linear Algebra Subroutine library (BLAS)</a></li>
<li><a href="http://www.netlib.org/lapack">Linear Algebra Package (LAPACK)</a></li>
<li><a href="https://github.com/opencollab/arpack-ng" target="_blank">ARPACK, a collection of Fortran77 subroutines designed to solve large scale
eigenvalue problems</a></li>
<li><a href="http://www.fftw.org/" target="_blank">Fastest Fourier Transform in the West (FFTW3)</a></li>
<li><a href="http://www.gnu.org/software/glpk" target="_blank">GNU Linear Programming Kit (GLPK)</a></li>
<li><a href="https://www.gnu.org/software/gperf/" target="_blank">GNU gperf, a perfect hash function generator</a></li>
<li><a href="http://www.hdfgroup.org/HDF5" target="_blank">HDF5, a program for reading Hierarchical Data Format files</a></li>
<li><a href="http://www.qhull.org/" target="_blank">Qhull, a computational geometry library</a></li>
<li><a href="http://sourceforge.net/projects/qrupdate" target="_blank">QRUPDATE, a QR factorization updating library</a></li>
<li><a href="http://faculty.cse.tamu.edu/davis/suitesparse.html" target="_blank">SuiteSparse, a sparse matrix factorization library</a></li>
</ul>

<h2>Folder structure</h2>

Some of the libraries above will be needed just for installation, and some will be needed in the final buildpack for Octave to run. The libraries that are needed for installation only can be installed in <span class="tt">/app/.heroku</span>, but the libraries that will go in the buildpack should be installed inside of Octave's install directory (which we have to make). SOmething like this will suffice:
{% highlight console %}
~ $ cd /app/.heroku
~/.heroku $ mkdir octave
~/.heroku $ cd octave
~/.heroku/octave $ mkdir deps
{% endhighlight %}
We will use this folder <span class="tt">/app/.heroku/octave/deps</span> to install the libraries that should end up in the final buildpack.

<h2>BLAS and LAPACK (with ATLAS)</h2>

Instead of installing the normal BLAS and LAPACK, we'll use an accelerated BLAS library (that will hopefully make Octave run a little faster) called <a href="http://math-atlas.sourceforge.net/">ATLAS</a>. This has to be done first, as some of the other libraries will need to refer to BLAS and LAPACK.

We'll do this inside the <span class="tt">../octave/deps</span> folder. ATLAS is a bit unusual because the install folder must be different than the source folder and the build folder. We must also have a tar-file for LAPACK saved somewhere and tell ATLAS where this is (so that it can build its own LAPACK library).

The ATLAS tar-file can be found <a href="http://sourceforge.net/projects/math-atlas/" target="_blank">here</a>, and the LAPACK tar-file can be found <a href="http://www.netlib.org/lapack/lapack-3.5.0.tgz" target="_blank">here</a>. The former must be scp'd onto the Heroku dyno, and the latter can be curl'd. Let's assume that both of these files are in the <span class="tt">../octave/deps</span> folder.  We also need to tell ATLAS to build shared libraries with the <span class="tt">--shared</span> command.
{% highlight console %}
~/.heroku/octave/deps $ tar -xjf atlas3.10.2.tar.bz2
~/.heroku/octave/deps $ mv ATLAS atlas-src
~/.heroku/octave/deps $ mkdir atlas-bld && mkdir atlas
~/.heroku/octave/deps $ cd atlas-bld
~/.heroku/octave/deps/atlas-bld $ ../atlas-src/configure --prefix=/app/.heroku/octave/deps/atlas --shared --with-netlib-lapack-tarfile=/app/.heroku/octave/deps/lapack-3.5.0.tgz
~/.heroku/octave/deps/atlas-bld $ make && make install
{% endhighlight %}

All other libraries that need to find the BLAS or LAPACK libraries should point to a specific s-file: <span clas="tt">/app/.heroku/octave/deps/atlas/lib/libtatlas.so</span>.

<h2>ARPACK</h2>

ARPACK is thankfully maintained in GitHub these days and is fairly easy to install. We only need to clone the GitHub repository and point the configure script to our BLAS library that was installed above. The GitHub command line utilities are already installed on a Heroku dyno, making the operation very simple.
{% highlight console %}
~/.heroku/octave/deps $ git clone https://github.com/opencollab/arpack-ng.git
~/.heroku/octave/deps $ mkdir arpack
~/.heroku/octave/deps $ cd arpack-ng
~/.heroku/octave/deps/arpack-ng $ sh bootstrap
~/.heroku/octave/deps/arpack-ng $ ./configure --prefix=/app/.heroku/octave/deps/arpack --with-blas=/app/.heroku/octave/deps/atlas/lib/libtatlas.so
~/.heroku/octave/deps/arpack-ng $ make && make install
{% endhighlight %}

Lastly, we must add the <span class="tt">lib</span> directory to <span class="tt">LDFLAGS</span> and <span class="tt">LD_LIBRARY_PATH</span>.
{% highlight console %}
~/.heroku/octave/deps/arpack-ng $ cd ../arpack 
~/.heroku/octave/deps/arpack $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$PWD/lib
~/.heroku/octave/deps/arpack $ export LDFLAGS="$LDFLAGS -L$PWD/lib"
{% endhighlight %}

<h2>FFTW</h2>

The Fastest Fourier Transform in the West library has a wicked cool name but was a big pain to figure out how to install. This must be installed twice, once with a single-precision/float version (which we will call FFTWF) and once with a double-precision version (which we will call FFTW). FFTWF must be installed in the <span class="tt">../octave/deps</span> folder, while FFTW will be installe<span class="tt">lib</span> in <span class="tt">/app/.heroku</span> as it is not needed in the final buildpack. 

For both sets of libraries, we will need to install shared libraries with the <span class="tt">--enable-shared</span> option. First, I'll go over FFTWF.
{% highlight console %}
~/.heroku/octave/deps $ curl ftp://ftp.fftw.org/pub/fftw/fftw-3.3.4.tar.gz -s -O
~/.heroku/octave/deps $ tar -xzf fftw-3.3.4.tar.gz
~/.heroku/octave/deps $ mkdir fftwf
~/.heroku/octave/deps $ cd fftw-3.3.4
~/.heroku/octave/deps/fftw-3.3.4 $ ./configure --prefix=/app/.heroku/octave/deps/fftwf --enable-float --enable-shared
~/.heroku/octave/deps/fftw-3.3.4 $ make && make install
{% endhighlight %}

We must add the <span class="tt">lib</span> and <span class="tt">include</span> directories to <span class="tt">LDFLAGS</span> and <span class="tt">CPPFLAGS</span>, respectively.
{% highlight console %}
~/.heroku/octave/deps/fftw-3.3.4 $ cd ../fftwf
~/.heroku/octave/deps/fftwf $ export LDFLAGS="$LDFLAGS -L$PWD/lib"
~/.heroku/octave/deps/fftwf $ export CPPFLAGS="$CPPFLAGS -I$PWD/include"
{% endhighlight %}

FFTW has a similar procedure, but in a different folder and without the <span class="tt">--enable-float</span> option.
{% highlight console %}
~/.heroku $ curl ftp://ftp.fftw.org/pub/fftw/fftw-3.3.4.tar.gz -s -O
~/.heroku $ tar -xzf fftw-3.3.4.tar.gz
~/.heroku $ mkdir fftw
~/.heroku $ cd fftw-3.3.4
~/.heroku/fftw-3.3.4 $ ./configure --prefix=/app/.heroku/fftw --enable-shared
~/.heroku/fftw-3.3.4 $ make && make install
{% endhighlight %}

Once again, update the flags.
{% highlight console %}
~/.heroku/fftw-3.3.4 $ cd ../fftw
~/.heroku/fftw $ export LDFLAGS="$LDFLAGS -L$PWD/lib"
~/.heroku/fftw $ export CPPFLAGS="$CPPFLAGS -I$PWD/include"
{% endhighlight %}

<h2>GLPK</h2>

GLPK is a very straightforward install, as you can get it from the GNU FTP server and it needs no special configuration. It can go directly into <span class="tt">/app/.heroku</span>. Without further comment...
{% highlight console %}
~/.heroku $ curl ftp://ftp.gnu.org/gnu/glpk/glpk-4.55.tar.gz -s -O
~/.heroku $ tar -xzf glpk-4.55.tar.gz
~/.heroku $ mkdir glpk
~/.heroku $ cd glpk-4.55
~/.heroku/glpk-4.55 $ ./configure --prefix=/app/.heroku/glpk
~/.heroku/glpk-4.55 $ make && make install
{% endhighlight %}

Once again, update the flags.
{% highlight console %}
~/.heroku/glpk-4.55 $ cd ../glpk
~/.heroku/glpk $ export LDFLAGS="$LDFLAGS -L$PWD/lib"
~/.heroku/glpk $ export CPPFLAGS="$CPPFLAGS -I$PWD/include"
{% endhighlight %}

<h2>GNU gperf</h2>

GNU gperf is another pretty one to install. We can put it in the <span class="tt">/.heroku</span> folder, and it follows the standard configure/make/install procedure. Once it has been installed, add the <span class="tt">\bin</span> directory to the path.

{% highlight console %}
~/.heroku $ curl http://ftp.gnu.org/pub/gnu/gperf/gperf-3.0.4.tar.gz -s -O
~/.heroku $ tar -xzf gperf-3.0.4.tar.gz
~/.heroku $ mkdir gperf
~/.heroku $ cd gperf-3.0.4
~/.heroku/gperf-3.0.4 $ ./configure --prefix=/app/.heroku/gperf
~/.heroku/gperf-3.0.4 $ make && make install
~/.heroku/gperf-3.0.4 $ cd ../gperf/bin
~/.heroku/gperf-3.0.4 $ export PATH=$PATH:$PWD
{% endhighlight %}

<h2>HDF5</h2>

HDF5 is another very easy one. The only trick is that it must be installed in <span class="tt">../octave/deps</span> as it needs to be included as part of the final buildpack.
{% highlight console %}
~/.heroku/octave/deps $ curl http://www.hdfgroup.org/ftp/HDF5/current/src/hdf5-1.8.14.tar.gz -s -O 
~/.heroku/octave/deps $ tar -xzf hdf5-1.8.14.tar.gz 
~/.heroku/octave/deps $ mkdir hdf5
~/.heroku/octave/deps $ cd hdf5-1.8.14
~/.heroku/octave/deps/hdf5-1.8.14 $ ./configure --prefix=/app/.heroku/octave/deps/hdf5
~/.heroku/octave/deps/hdf5-1.8.14 $ make && make install 
{% endhighlight %}

As before, update the flags.
{% highlight console %}
~/.heroku/octave/deps/hdf5-1.8.14 $ cd ../hdf5
~/.heroku/octave/deps/hdf5 $ export LDFLAGS="$LDFLAGS -L$PWD/lib"
~/.heroku/octave/deps/hdf5 $ export CPPFLAGS="$CPPFLAGS -I$PWD/include"
{% endhighlight %}

<h2>Qhull</h2>

Qhull is one of the trickier libraries to install because it doesn't follow the standard configure/make/install procedure. Once we unzip the tar file, we'll need to edit the Makefile to specify our install directory. I hate using whatever editor comes loaded on a Heroku dyno, so I actually did this locally and then scp'd the Makefile back onto the Heroku dyno after it was edited. We can install it in <span class="tt">/app/.heroku</span>.
{% highlight console %}
~/.heroku $ curl http://www.qhull.org/download/qhull-2012.1-src.tgz -s -O
~/.heroku $ tar -xzf qhull-2012.1-src.tgz
~/.heroku $ mkdir qhull
~/.heroku $ cd qhull-2012.1
{% endhighlight %}

Note that there is no configure script here, which is why we have to edit the Makefile. Edit the Makefile on line 75 so that it reads:
{% highlight emacs %}
DESTDIR = /app/.heroku/qhull
{% endhighlight %}

Now we can actually perform the installation.
{% highlight console %}
~/.heroku/qhull-2012.1 $ make && make install 
{% endhighlight %}

In addition to adding the <span class="tt">include</span> directory to <span class="tt">CPPFLAGS</span>, we must add the <span class="tt">lib</span> directory to <span class="tt">LDFLAGS</span> and <span class="tt">LD_LIBRARY_PATH</span>.
{% highlight console %}
~/.heroku/qhull-2012.1 $ cd ../qhull
~/.heroku/qhull $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$PWD/lib
~/.heroku/qhull $ export LDFLAGS="$LDFLAGS -L$PWD/lib"
~/.heroku/qhull $ export CPPFLAGS="$CPPFLAGS -I$PWD/include"
{% endhighlight %}

<h2>QRUPDATE</h2>

QRUPDATE cannot be curl'd; it must be scp'd from <a href="http://sourceforge.net/projects/qrupdate/?source=typ_redirect" target="_blank">here</a>. This is another tough one, since it does not use the configure/make/make install procedure. Once you have the source code loaded on the Heroku dyno, you must edit the <span class="tt">Makeconf</span> file to point to our BLAS and LAPACK libraries as well as our non-standard install directory (inside of <span class="tt">../octave/deps</span>).
{% highlight console %}
~/.heroku/octave/deps $ tar -xzf qrupdate-1.1.2.tar.gz 
~/.heroku/octave/deps $ mkdir qrupdate
~/.heroku/octave/deps $ cd qrupdate-1.1.2
{% endhighlight %}

Edit the Makeconf file to point to the proper directories.
{% highlight emacs %}
BLAS=/app/.heroku/octave/deps/atlas/lib/libtatlas.so
LAPACK=/app/.heroku/octave/deps/atlas/lib/libtatlas.so
PREFIX=/app/.heroku/octave/deps/qrupdate
{% endhighlight %}

Now we can go back into our install folder and tell QRUPDATE to make and install shared libraries.
{% highlight console %}
~/.heroku/octave/deps/qrupdate-1.1.2 $ make solib
~/.heroku/octave/deps/qrupdate-1.1.2 $ make install
{% endhighlight %}

The <span class="tt">lib</span> directory must be added to <span class="tt">LD_LIBRARY_PATH</span> and <span class="tt">LDFLAGS</span>.
{% highlight console %}
~/.heroku/octave/deps/qrupdate-1.1.2 $ cd ../qrupdate
~/.heroku/octave/deps/qrupdate $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$PWD/lib
~/.heroku/octave/deps/qrupdate $ export LDFLAGS="$LDFLAGS -L$PWD/lib"
{% endhighlight %}

<h2>SuiteSparse</h2>

SuiteSparse is the last library to install. We can put it right into <span class="tt">/app/.heroku</span>, but it's a bit of a pain to install (this should sound familiar by now). For starters, we must use a version less than 4.3 as the build configuration changed and will not work with Octave. SuiteSparse also does not follow the configure/make/make install procedure--we must edit a Makefile to point to our BLAS/LAPACK and install directories.

To begin with, download the source and set up the folder structure. Our install folder will be <span class="tt">/app/.heroku/SuiteSparse</span>, and we need to create a <span class="tt">lib</span> directory and <span class="tt">include</span> directory before we actually run the installation scripts.
{% highlight console %}
~/.heroku/ $ curl http://faculty.cse.tamu.edu/davis/SuiteSparse/SuiteSparse-4.2.1.tar.gz -s -O
~/.heroku/ $ tar -xzf SuiteSparse-4.2.1.tar.gz
~/.heroku/ $ mv SuiteSparse SuiteSparse-source
~/.heroku/ $ mkdir SuiteSparse
~/.heroku/ $ cd SuiteSparse
~/.heroku/SuiteSparse $ mkdir lib && mkdir include
{% endhighlight %}

We must now edit a the file <span class="tt">SuiteSparse_config/SuiteSparse_config.mk</span> and make it point to the right directories.
{% highlight emacs %}
INSTALL_LIB = /app/.heroku/SuiteSparse/lib
INSTALL_INCLUDE = /app/.heroku/SuiteSparse/include
BLAS = /app/.heroku/octave/deps/atlas/lib/libtatlas.so
LAPACK = /app/.heroku/octave/deps/atlas/lib/libtatlas.so
{% endhighlight %}

Now we can get back into our install directory and tell SuiteSparse to build and install its libraries.
{% highlight console %}
~/.heroku/SuiteSparse-source $ make library
~/.heroku/SuiteSparse-source $ make install
{% endhighlight %}

Once again, update the flags.
{% highlight console %}
~/.heroku/SuiteSparse-source $ cd ../SuiteSparse
~/.heroku/SuiteSparse $ export LDFLAGS="$LDFLAGS -L$PWD/lib"
~/.heroku/SuiteSparse $ export CPPFLAGS="$CPPFLAGS -I$PWD/include"
{% endhighlight %}

<h2>Configure and Install Octave</h2>

Now that all the required libraries are installed, we can finally install Octave. The source code is hosted on the GNU FTP server and we can use the normal configure/make/install process. We only need to specify where our BLAS/LAPACK libraries are, and we need to tell Octave not to install any of the graphics components (since we will be using this as a back-end computational tool with no graphics required).

We will perfrom the installation in <span class="tt">/app/.heroku</span> as follows.
{% highlight console %}
~/.heroku/ $ curl ftp://ftp.gnu.org/gnu/octave/octave-3.8.2.tar.gz -s -O
~/.heroku/ $ tar -xzf octave-3.8.2.tar.gz
~/.heroku/ $ cd octave-3.8.2
~/.heroku/octave-3.8.2 $ ./configure --prefix=/app/.heroku/octave --disable-gui --disable-docs --disable-java --without-opengl --with-blas=/app/.heroku/octave/deps/atlas/lib/libtatlas.so --with-lapack=/app/.heroku/octave/deps/atlas/lib/libtatlas.so
~/.heroku/octave-3.8.2 $ make && make install  
{% endhighlight %}

The configure script will take about 10 minutes to run, and the actuall installation will take a few hours. Once it has finished up, make sure to add the <span class="tt">bin</span> directory to the <span class="tt">PATH</span> environment variable so that the system knows where to look for Octave (since it is installed in a non-standard directory).
{% highlight console %}
~/.heroku/octave-3.8.2 $ cd ../octave
~/.heroku/octave $ export PATH=$PWD/bin:$PATH 
{% endhighlight %}

For the purposes of making a buildpack, we will want to compress the installed Octave directory into a tarball and scp it to a permanent storage space. I'll go over how to turn this into a buildpack for Heroku in the next post.