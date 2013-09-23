---
title: "Syncing the OPM build system"
layout: post
date: 2013-09-23 11:02 +0200
tags:
- OPM
- Git
---
In this post, I am going to show how I keep files in the `cmake` subdirectory synchronized across two projects.


To do this I am going to "spool" all new changes from the master repository over to a list which is then applied to the "slave". I do this in a separate branch which is then merged, so it looks just like a pull request in GitHub (except that it never appears there). You could probably get the same effect with the newer command `git subtree split --rejoin` but that'll have to be for later post.

I am going to assume that the two projects are named opm-core for the master and opm-foo for the slave. You will of course have to substitute in the right names for your own repository.

So, without further ado, let's get started:

<ol>
<li> We need to find the last common point that was synchronized. If you don't add anything offhand to opm-foo this will simply be the last commit that was added to the `cmake` directory there:

{% highlight sh %}
git log --no-merges --format="%s" -n 1 cmake/
{% endhighlight %}

The SHA isn't of much value to us (since it is specific for that project), so we just print out the title. If you use generic titles such as "Made some changes" I hope this was a valuable lesson for why that is a bad idea.

If you have added some extra commits only in opm-foo, then you'll have to print more than one commit and work your way down the list until you find one that is common with opm-core.

<li> Next, we'll have to find this commit back in opm-core. We can do this by searching after the title in printed in the previous step.
{% highlight sh %}
( cd ../opm-core ; git log --oneline -F --grep "Print version number" cmake/ )
{% endhighlight %}

The default output of the --oneline format is both the SHA and the title; we carry on with the SHA, which should look something like  `976c5c54`.

<li> Now we run a script to generate import statements for everything from this particular commit:
{% highlight sh %}
( cd ../opm-core;
git log --no-merges --topo-order --reverse --format='%H %s' 976c5c54..master -- cmake/ |
while read commit message; do
    echo -e "# $message\nwget -q -O - https://github.com/opm/opm-core/commit/$commit.patch | git am -3 -\n"
done
) >> ../cmake_01.sh
{% endhighlight %}

You can of course wrap this into a script that takes the SHA as a parameter. I have. :-)

Let's dissect it: The `git log` command generates a log of all commits in the `cmake/` directory that has happened *after* the one with the SHA. By asking for topological order, we get those in the same pull request grouped together instead of interspersed by date. We want order reversed, because we are going to commit the earliest first.

The `while` loop formats the output of each commit; first it prints the title as a comment so we can see which commit it was supposed to by -- SHAs aren't particularily fun to memorize, and the next is a command to download this patch from GitHub using `wget` and apply it into the current Git repository with `git am`.

The output of the script goes into a file named `cmake_XX.sh` in the parent directory. This makes it possible to edit the script separately if any of the patches need some massaging before they'll commit (removing parts that aren't in `cmake/` for instance). I am also going to use this name for the branch in the following. Remember to keep some kind of counter in the name here, to avoid confusion later.

<li> Turning our attention back to opm-foo, we create a new branch to hold this patch series:
{% highlight sh %}
git checkout -b cmake_01 master
{% endhighlight %}

<li> Then we run the generated script in this branch:
{% highlight sh %}
bash -e -x ../cmake_01.sh
{% endhighlight %}

The `-e` parameter to bash is to have it stop after the first error. If a patch doesn't apply, it may be because you may have applied it already. Figure out which line this is in the script and remove it, then do `git am --abort` and try again.

If it doesn't apply because you have introduced commits in opm-foo that conflict with the mainline, then I wish you the very best of luck, soldier: You're gonna need it.

<li> Merge this into master:
{% highlight sh %}
git checkout master
git merge -no-ff cmake_01
{% endhighlight %}

The `-no-ff` option will make the merge look like a pull request on GitHub. I am usually not very keen on "merge bubbles", but I reckon it is a good idea to have them here since the evolution of the build system really is a separate feature from the rest.

<li> Verify that we are now good by comparing our `cmake/` directory to the original (apart from any lingering backup files):
{% highlight sh %}
diff -q -r ../opm-core/cmake ../opm-foo/cmake | grep -v ~$
{% endhighlight %}

<li> And then finally we push this to the main repo!
{% highlight sh %}
git push upstream master
{% endhighlight %}

Only the maintainers should do this as there won't be any notifications through the GitHub system that the change was done.