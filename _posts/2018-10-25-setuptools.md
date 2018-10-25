---
title: "Upgrading Python setuptools on Debian/Ubuntu"
layout: post
date: 2018-10-25 12:30 +0200
tags:
- python
- ubuntu 
--- 

Say that your system is still on Ubuntu Xenial 16.04. From time to
time you may have encountered this problem: There is a Python package
that you want to install, but it turns out that this requires a newer
version of setuptools than you have installed on your system.

No problem! you think; I'll simply issue the command:

{% highlight sh %}
python3 -m pip install --upgrade setuptools
{% endhighlight %}

in my virtualenv first. This is of course prompty executed without any
apparent problems, so you proceed back to the installation of your
package. However, to your chagrin this ends up with the
oh-not-so-helpful error message ""No module named
'pkg_resources.py31compat'". Your puzzlement is perhaps even greater
when you check /usr/local/lib/python3.5/dist-packages, where all your
own globally pip-installed packages will end up, and figure out that
there actually *is* a py31compat directory! What is going on!?

It turns out that the Debian mainainers, in their infinite wisdom, has
patched pip, so that it executes in a "isolated" environment where
your regular packages are shoved aside, and replaced with the ones
that are present in /usr/share/python-wheels. Thus, you always get the
old packages whenever you run pip, even though everything looks nice
and dandy on the outside.

This is the reason why you as a rule-of-thumb should not install newer
versions of Python packages with pip, that you have already installed
system version of with apt. Sometimes, when you are in setup.py
scripts, you'll see the old version; otherwise you'll not.

Fixing this is not just a matter of diverting away the obsolete
wheels, but is rather a bit convoluted, because venv, the package that
sets up virtualenv, calls a script named ensurepip to "install" pip in
your virtualenv, and this also runs in isolated mode. This script,
amongst many other things, copy the wheels from
/usr/share/python-wheel into a a sub-directory share/python-wheels in
your virtualenv directory, and it is a bit picky about how the source
directory looks like. There should for instance only be one version of
each wheel, and it has a predefined list of wheels that it looks
for. If there is any errors, the exceptions that are raised are
typically squashed and only a generic message about ensurepip is
shown. (You can debug by running the command that it list as failed,
though).

Furthermore, the pkg_resources wheel that is present does not really
exist as its own package. The files inside that wheel is usually
present is newer setuptools. So, we must create a dummy pkg_resource
wheel for ensurepip to install.

OK, enough talk; let's get down to some actual command-lines! We start
out our endeavor by actually installing the system version of venv and
pip, so that these don't get pulled in later and undo all our work:

{% highlight sh %}
sudo apt-get install -y python3-venv python3-pip
{% endhighlight %}

Then we create temporary directory where we'll prepare all the new
stuff that's going in:

{% highlight sh %}
TEMP_DIR=$(mktemp -d)
cd "${TEMP_DIR}"
{% endhighlight %}

We start out by making an empty dummy package for pkg_resources:

{% highlight sh %}
PKG_DIR=pkg_resources-0.0.0.dist-info
mkdir -p $PKG_DIR
cat > $PKG_DIR/metadata.json <<'EOF'
{"extensions": {"python.details": {"document_names": {"description": "DESCRIPTION.rst"}}}, "generator": "bdist_wheel (0.29.0)", "metadata_version": "2.0", "name": "pkg_resources", "summary": "UNKNOWN", "version": "0.0.0"}
EOF
cat > $PKG_DIR/METADATA <<'EOF'
Metadata-Version: 2.0
Name: pkg_resources
Version: 0.0.0
Summary: UNKNOWN
Home-page: UNKNOWN
Author: UNKNOWN
Author-email: UNKNOWN
License: UNKNOWN
Platform: UNKNOWN

UNKNOWN


EOF
cat > $PKG_DIR/DESCRIPTION.rst <<'EOF'
UNKNOWN


EOF
cat > $PKG_DIR/WHEEL <<'EOF'
Wheel-Version: 1.0
Generator: bdist_wheel (0.29.0)
Root-Is-Purelib: true
Tag: py2-none-any
Tag: py3-none-any

EOF
for f in DESCRIPTION.rst METADATA WHEEL metadata.json; do
    printf "%s,%s,%d\n" "$PKG_DIR/$f" $(sha256sum $PKG_DIR/$f | cut -d' ' -f 1) $(stat --printf='%s' $PKG_DIR/$f) >> $PKG_DIR/RECORD
done
printf "%s,,\n" $PKG_DIR/RECORD >> $PKG_DIR/RECORD
zip -r pkg_resources-0.0.0-py2.py3-none-any.whl $PKG_DIR
{% endhighlight %}

And then we download a new version of setuptools:

{% highlight sh %}
python3 -m pip download setuptools
{% endhighlight %}

Everything should now be ready. We can move aside the old,
system-installed wheels by setting up diversions:

{% highlight sh %}
for pkg in setuptools pkg_resources; do
  for whl in $(find /usr/share/python-wheels -name "$pkg-*.whl"); do
    sudo dpkg-divert --local --divert $whl.distrib --add $whl
	sudo mv $whl $whl.distrib
  done
done
{% endhighlight %}

Next, we install our dummy replacement for pkg_resources:

{% highlight sh %}
sudo install -o root -g root -m 644 -t /usr/share/python-wheels $(ls -1b pkg_resources-*.whl | head -n 1)
{% endhighlight %}

For setuptools itself, we install it to a non-distribution system
directory, and setup links to this. If you like you can give someone
else write access to the *directory* /usr/local/share/python-wheels,
and they can then add new versions, and point the symbolic link to an
even newer version, without touching the rest of the setup. We then
put a link in /usr/share/python-wheels to this setup; this is
reminiscent of the way programs in /usr/bin points to a link in
/etc/alternatives, which again points to a more specific version.

{% highlight sh %}
SETUPTOOLS=$(ls -1b setuptools-*.whl | head -n 1)
sudo install -o root -g root -d /usr/local/share/python-wheels
sudo install -o root -g root -m 644 -t /usr/local/share/python-wheels $SETUPTOOLS
sudo ln -s $SETUPTOOLS /usr/local/share/python-wheels/setuptools-0.0.0-py2.py3-none-any.whl
sudo ln -s /usr/local/share/python-wheels/setuptools-0.0.0-py2.py3-none-any.whl /usr/share/python-wheels/setuptools-0.0.0-py2.py3-none-any.whl
{% endhighlight %}

Finally, our setup is complete and we clean up our temporary directory:

{% highlight sh %}
cd -
rm -rf "${TEMP_DIR}"
{% endhighlight %}

