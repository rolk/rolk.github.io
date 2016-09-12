You can use this repository as a template to create a blog on [GitHub Pages](http://pages.github.com).

Setting up your own blog
========================

1. [Create the repository](https://help.github.com/articles/create-a-repo) "username.github.io" on GitHub (yes, the domain name is supposed to be part of the repo name (and also yes, you are supposed to substitute `username` with your own GitHub username)).

2. Clone the [Jekyll Bootstrap](http://jekyllbootstrap.com/) repository:
```sh
git clone http://github.com/rolk/jekyll-bootstrap username.github.io
```

3. Change the five fields title, tagline, author name and email and baseurl in the file `_config.yml`:
```
title : Jekyll Bootstrap
tagline: Site Tagline
author :
  name : Name Lastname
  email : blah@email.test
baseurl : http://username.github.io
```
	
4. Connect the local clone of Jekyll Bootstrap to your newly created repository:
```sh
git remote rm origin
git remote add origin ssh://git@github.com/username/username.github.io
```

5. Push the files online:
```sh
git push origin master
```

After 10 minutes or so, the blog will be ready at [http://username.github.io](http://username.github.io).

Writing posts
=============

1. Create new file names `_posts/yyyy-mm-dd-subject.md` (substituting of course `yyyy-mm-dd` with today's date and `subject` with something more sensible).

2. Use this header:
```
---
title: "Something"
layout: post
date: yyyy-mm-dd hh:ss +0200
tags:
- Foo
- Bar
---
```

3. Write the rest of the blog in [Markdown](http://daringfireball.net/projects/markdown/syntax) syntax.

4. If you need to embed some programming code, do like this:
```
{% highlight c++ linenos=table %}
...
{% endhighlight %}
```

5. Add this file to the repo and push!
```sh
git add _posts/yyyy-mm-dd-subject.md
git commit
git push origin master
```

Testing locally
===============
Install the Jekyll Ruby-gem and support plug-ins:

```sh
sudo sh -c 'umask 022; gem install jekyll jekyll-paginate kramdown rouge'
```

Now that you have Jekyll installed locally, you can test the site by issuing the command:
```sh
jekyll serve --watch --baseurl ""
```

and then browse to `http://localhost:4000`.
