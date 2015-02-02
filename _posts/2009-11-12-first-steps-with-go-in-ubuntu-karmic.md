---
title: First steps with Go in Ubuntu Karmic
author: Carlos Vara
layout: post
permalink: /2009/11/first-steps-with-go-in-ubuntu-karmic/
categories:
  - go
tags:
  - go
  - programming
---
It&#8217;s all over the Internet today, Google has released a new programming language called <a href="http://golang.org" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://golang.org']);">Go</a>. As of what I have been able to see so far, Go can be described as a systems language which aims to leverage Python&#8217;s expressiveness into the grounds of compiled languages as C++. Also, Go is both fast to compile and to execute. You can see some pretty impressive footage of <a href="http://www.youtube.com/watch?v=wwoWei-GAPo" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://www.youtube.com']);">Go&#8217;s compiler speed in this video</a>.

A good starting point for Go is that it&#8217;s Open Source, so you can start playing with it right now (as long as you are a linux or mac user). That&#8217;s what I have done, and I&#8217;m posting my first steps under Ubuntu Karmic in case it can help anybody.

### Installation

Go&#8217;s compiler and tools are currently distributed in source, so you will first need to ensure that you have certain compiling packages installed in your system so you will be able to compile Go&#8217;s tools. Also, Google uses Mercurial for revision control, so you will need it for checking out Go&#8217;s tree.

Execute the following in a shell:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="bash" style="font-family:monospace;"><span style="color: #c20cb9; font-weight: bold;">sudo</span> <span style="color: #c20cb9; font-weight: bold;">apt-get install</span> mercurial <span style="color: #c20cb9; font-weight: bold;">bison</span> <span style="color: #c20cb9; font-weight: bold;">gcc</span> libc6-dev <span style="color: #c20cb9; font-weight: bold;">ed</span></pre>
      </td>
    </tr>
  </table>
</div>

And you will have all what you need to fetch and build Go&#8217;s sources. We will install it in a clean way so everything is under one directory and maintenance and an eventual removal is easy. First, you need to create the directory where everything Go-related will go:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="bash" style="font-family:monospace;"><span style="color: #c20cb9; font-weight: bold;">mkdir</span> ~<span style="color: #000000; font-weight: bold;">/</span>golang</pre>
      </td>
    </tr>
  </table>
</div>

Now, you need to edit your .bashrc file. It&#8217;s located at the root of your home directory. Add the following lines at its end (you will have to change the GOARCH variable to `amd64` if you are running the 64bit version):

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="bash" style="font-family:monospace;"><span style="color: #666666; font-style: italic;"># Go Installation</span>
<span style="color: #7a0874; font-weight: bold;">export</span> <span style="color: #007800;">GOROOT</span>=~<span style="color: #000000; font-weight: bold;">/</span>golang
<span style="color: #7a0874; font-weight: bold;">export</span> <span style="color: #007800;">GOOS</span>=linux
<span style="color: #7a0874; font-weight: bold;">export</span> <span style="color: #007800;">GOARCH</span>=<span style="color: #000000;">386</span>
<span style="color: #7a0874; font-weight: bold;">export</span> <span style="color: #007800;">GOBIN</span>=<span style="color: #800000;">${GOROOT}</span><span style="color: #000000; font-weight: bold;">/</span>bin
<span style="color: #007800;">PATH</span>=<span style="color: #800000;">${PATH}</span>:<span style="color: #800000;">${GOBIN}</span>
<span style="color: #7a0874; font-weight: bold;">export</span> PATH</pre>
      </td>
    </tr>
  </table>
</div>

With that in place, open a new terminal so your .bashrc gets reloaded. Within that terminal, check out Go&#8217;s sources with Mercurial by typing:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="bash" style="font-family:monospace;">hg clone <span style="color: #660033;">-r</span> release https:<span style="color: #000000; font-weight: bold;">//</span>go.googlecode.com<span style="color: #000000; font-weight: bold;">/</span>hg<span style="color: #000000; font-weight: bold;">/</span> <span style="color: #007800;">$GOROOT</span></pre>
      </td>
    </tr>
  </table>
</div>

And once it finishes checking out, create the `bin` directory and compile with:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="bash" style="font-family:monospace;"><span style="color: #c20cb9; font-weight: bold;">mkdir</span> <span style="color: #007800;">$GOROOT</span><span style="color: #000000; font-weight: bold;">/</span>bin
<span style="color: #7a0874; font-weight: bold;">cd</span> <span style="color: #007800;">$GOROOT</span><span style="color: #000000; font-weight: bold;">/</span>src
.<span style="color: #000000; font-weight: bold;">/</span>all.bash</pre>
      </td>
    </tr>
  </table>
</div>

If all went right, you are now ready to start using Go. You may check by typing 8g or 6g (for 386 and amd64 respectively) and seeing if it executes the compiler. I must say that the installation worked flawlessly for me following the instructions in Go&#8217;s homepage, so up to this point this guide is just an adaptation taking care of Ubuntu&#8217;s specific parts.

### Keeping Go up to date

Go&#8217;s development is quite lively. Roughly 10 minutes past installing, I checked the repository and there were 15 new updates. So it&#8217;s a good idea to have a recent version of Go installed (specially if you are going to do things as bug reporting).

You can keep your installation updated by executing:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="bash" style="font-family:monospace;"><span style="color: #7a0874; font-weight: bold;">cd</span> <span style="color: #007800;">$GOROOT</span>
hg pull <span style="color: #660033;">-u</span>
<span style="color: #7a0874; font-weight: bold;">cd</span> src
.<span style="color: #000000; font-weight: bold;">/</span>all.bash</pre>
      </td>
    </tr>
  </table>
</div>

Also, if you would want to remove Go from your system, it&#8217;s as easy as deleting the $GOROOT directory and removing the added lines from your .bashrc file.

### Start!

OK, now you are ready to start experimenting with Go. You can write the standard Hello World program as:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="go" style="font-family:monospace;"><span style="color: #b1b100; font-weight: bold;">package</span> main
&nbsp;
<span style="color: #b1b100; font-weight: bold;">import</span> format <span style="color: #cc66cc;">"fmt"</span>
&nbsp;
<span style="color: #993333;">func</span> main<span style="color: #339933;">()</span> <span style="color: #339933;">{</span>
	format<span style="color: #339933;">.</span>Printf<span style="color: #339933;">(</span><span style="color: #cc66cc;">"Hello World!<span style="color: #000099; font-weight: bold;">\n</span>"</span><span style="color: #339933;">)</span>
<span style="color: #339933;">}</span></pre>
      </td>
    </tr>
  </table>
</div>

And compile, link &#038; run with:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="bash" style="font-family:monospace;">8g hello.go
8l <span style="color: #660033;">-o</span> hello hello.8
.<span style="color: #000000; font-weight: bold;">/</span>hello</pre>
      </td>
    </tr>
  </table>
</div>

As side note, if you are interested in a system wide installation (instead of per-user, as I just described), change your $GOROOT declaration to somewhere where every user has read permissions (like `/opt/golang` or `/usr/local/golang`), and edit /etc/environment instead of your user&#8217;s .bashrc. Also, you will have to execute most commands with sudo as you will need write permissions under $GOROOT.