---
title: Maven, a first day guide
author: Carlos Vara
layout: post
permalink: /2009/11/maven-a-first-day-guide/
categories:
  - Java
tags:
  - Java
  - maven
---
This is the first chapter of a mini-guide that will try first to set clear what the purpose of <a title="Apache Maven" href="http://maven.apache.org/" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://maven.apache.org']);">Apache Maven</a> is, and then show you how you can use it in your Java projects.

I have always preferred to start learning by example and by doing things instead of by reading lengthy manuals (there will be time for that once I have started to get the feel of the technology). So, I will write this aimed to a learner like me, hoping that some of you also prefer learning it this way.

Anyways, the introduction is over, let&#8217;s get to the meat <img src="http://carinae.net/wp-includes/images/smilies/icon_smile.gif" alt=":-)" class="wp-smiley" />

### Maven

There are plenty of sites telling you <a title="What is Maven?" href="http://maven.apache.org/what-is-maven.html" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://maven.apache.org']);">what Maven is</a>, so I will tell you what you will usually use Maven for. When developing a Java project, you will use Maven to perform tasks such as building, packaging, deploying or testing your project. Then, why use it instead of any other tool? These are some of Maven strong points:

  * **All** information regarding how you want to perform all these tasks is centralized in a single file: pom.xml.
  * Convention over configuration. If you adhere to Maven&#8217;s conventions (for example, where to place your .java files), your pom.xml file will be very concise.
  * A nice plugin ecosystem. Maven will probably have a plugin that does that not so usual task you want to perform.
  * And finally, dependency control. One of its more known features, with Maven it&#8217;s very easy to configure and check what Jars you need for each stage (building, testing and running).

In case you have used Ant to build-test-deploy-etc. your project, Maven can probably be its substitute, provided that you find Maven&#8217;s way of doing the task more adequate.

### Installation

If you haven&#8217;t already, you may get Maven from here: <a title="Maven download & install" href="http://maven.apache.org/download.html" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://maven.apache.org']);">http://maven.apache.org/download.html</a>. There is also installation instructions for Windows and Unix-like systems.

A note to Linux users, even though you can probably get Maven from your distro&#8217;s packaging system, you may consider installing it standalone. At least in Debian/Ubuntu systems the package pulls a gazillion dependencies that you don&#8217;t need.

### The minimal pom.xml

OK, you have Maven installed, let&#8217;s see what it can do for you. Create a directory, and place this pom.xml file in it:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="xml" style="font-family:monospace;"><span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;project</span> <span style="color: #000066;">xmlns</span>=<span style="color: #ff0000;">"http://maven.apache.org/POM/4.0.0"</span></span>
<span style="color: #009900;">  <span style="color: #000066;">xmlns:xsi</span>=<span style="color: #ff0000;">"http://www.w3.org/2001/XMLSchema-instance"</span></span>
<span style="color: #009900;">  <span style="color: #000066;">xsi:schemaLocation</span>=<span style="color: #ff0000;">"http://maven.apache.org/POM/4.0.0</span>
<span style="color: #009900;">    http://maven.apache.org/maven-v4_0_0.xsd"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>
&nbsp;
  <span style="color: #808080; font-style: italic;">&lt;!-- Maven's POM version --&gt;</span>
  <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;modelVersion<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>4.0.0<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/modelVersion<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
&nbsp;
  <span style="color: #808080; font-style: italic;">&lt;!-- When packaging, a JAR file will be produced --&gt;</span>
  <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;packaging<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>jar<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/packaging<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
&nbsp;
  <span style="color: #808080; font-style: italic;">&lt;!-- And the file will be named my-jar.jar --&gt;</span>
  <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>my-jar<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
&nbsp;
  <span style="color: #808080; font-style: italic;">&lt;!-- This tells Maven how your jar should be archived, should you</span>
<span style="color: #808080; font-style: italic;">     want to use it as a dependency for another project --&gt;</span>
  <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>tld.testing<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
  <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>0.1-Alpha<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
&nbsp;
  <span style="color: #808080; font-style: italic;">&lt;!-- The projects name --&gt;</span>
  <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;name<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>My Project<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/name<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
&nbsp;
<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/project<span style="color: #000000; font-weight: bold;">&gt;</span></span></span></pre>
      </td>
    </tr>
  </table>
</div>

It basically tells Maven that it should create a JAR file. You may now execute `mvn package` in the directory and you will see Maven create the requested JAR file in the also created target directory. Of course, the JAR will be empty apart from an auto-generated Manifest as there are no Java source files yet.

Take care, if it&#8217;s the first time you execute a Maven&#8217;s stage, <a title="Maven &quot;download the internet&quot;" href="http://www.google.com/search?hl=en&q=maven+&quot;download+the+internet&quot;&btnG=Search&meta=&aq=f&oq=" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://www.google.com']);">it will download the internet</a> (everything will be saved locally, so next time you execute it, probably nothing will have to be downloaded).

### Adding files to compile

If you have read Maven&#8217;s output to the last command, it will have told you that it had no files to compile. So, lets add a simple Java file, called App.java.

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #000000; font-weight: bold;">package</span> <span style="color: #006699;">mypkg</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> App <span style="color: #009900;">&#123;</span>
&nbsp;
  <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">static</span> <span style="color: #000066; font-weight: bold;">void</span> main<span style="color: #009900;">&#40;</span><span style="color: #003399;">String</span><span style="color: #009900;">&#91;</span><span style="color: #009900;">&#93;</span> args<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
    <span style="color: #003399;">System</span>.<span style="color: #006633;">out</span>.<span style="color: #006633;">println</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Hello Maven!"</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
  <span style="color: #009900;">&#125;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

You will have to create the directory `src/main/java/mypkg` and place it in there. Once it&#8217;s done, type again `mvn package` in the root of your project. You can check the target directory and see your compiled class in there under the classes directory, and the updated Jar file with the App.class now inside.

As a final check of this stage, execute `mvn exec:java -Dexec.mainClass="mypkg.App"`. This tells Maven to execute your compiled class, so you will see if everything is OK (you should see the &#8220;Hello Maven!&#8221; output between Maven&#8217;s info messages).

As you have seen, by adhering to Maven&#8217;s convention of where the source files should be placed, no extra configuration has been needed in the pom.xml file.

### Testing stage and the first dependency

If you read Maven&#8217;s output when packaging, you may have noticed a &#8220;No tests to run&#8221; output between the compiling and packaging step. In Maven&#8217;s way of doing things, there is a test stage between compiling and packaging. That means, once it has compiled your project, Maven will run any unit tests that you have against the compiled files, and in case it succeeds, it will procede to package. That&#8217;s a good thing in my book, so let&#8217;s give Maven a test to run.

Suppose we want to run a simple unit test coded in JUnit. We will need the JUnit JAR in the classpath, but only during the test stage. It&#8217;s now time to start using Maven&#8217;s dependency control, so we add the following before the `</project>` closing tag in the pom.

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="xml" style="font-family:monospace;">  <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;dependencies<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
      <span style="color: #808080; font-style: italic;">&lt;!-- Group and artifact id tell Maven where to look </span>
<span style="color: #808080; font-style: italic;">        for a dependency --&gt;</span>
      <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>junit<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
      <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>junit<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
      <span style="color: #808080; font-style: italic;">&lt;!-- And the version completes the information so it </span>
<span style="color: #808080; font-style: italic;">        knows exactly what JAR it must download --&gt;</span>
      <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>3.8.1<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
      <span style="color: #808080; font-style: italic;">&lt;!-- We want JUnit only in the test stage --&gt;</span>
      <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;scope<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>test<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/scope<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
  <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/dependencies<span style="color: #000000; font-weight: bold;">&gt;</span></span></span></pre>
      </td>
    </tr>
  </table>
</div>

With that information, next time you package your project Maven will automatically download JUnit JAR and add it in the classpath during your tests.

So let&#8217;s see if that works. Create a very simple (and not useful at all) JUnit test in a file called AppTest.java:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">junit.framework.TestCase</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> AppTest <span style="color: #000000; font-weight: bold;">extends</span> TestCase <span style="color: #009900;">&#123;</span>
&nbsp;
  <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> testSum<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #000000; font-weight: bold;">throws</span> <span style="color: #003399;">Exception</span> <span style="color: #009900;">&#123;</span>
    assertEquals<span style="color: #009900;">&#40;</span><span style="color: #cc66cc;">2</span>, <span style="color: #cc66cc;">1</span><span style="color: #339933;">+</span><span style="color: #cc66cc;">1</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
  <span style="color: #009900;">&#125;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

And place that file in `src/test/java/mypkg`. Then, execute `mvn package` again. You will see how Maven performs the test, outputs the report of the test stage and proceeds to package as there were no test failures.

With a minimal pom.xml, you have configured Maven to compile, test and package your project. That pom.xml file and directory structure would be a good starting point template, but Maven has a better solution than the copy pasting of that structure&#8230;

### The quickstart archetype

With Maven you can use lots of predefined *archetypes* that act like templates when starting a Maven managed project. This is very handy for when you are starting a WAR project or have some configuration that requires a predetermined configuration in the pom and file structure.

Open a terminal in a directory other than the one in which you did your first project, and:

  1. Execute: mvn archetype:generate
  2. Select maven-archetype-quickstart, it&#8217;s 15 on my list.
  3. Define value for groupId: : *tld.testing*
  4. Define value for artifactId: : *my-jar*
  5. Define value for version: 1.0-SNAPSHOT: : *0.1-Alpha*
  6. Define value for package: tld.testing: : *mypkg*

Once you finish, Maven will create a my-jar directory in which you will have an equivalent pom and directory structure to the one you hand-made in the previous sections. The good thing is now **you know why** it defines those things, and have a better understanding of Maven than if you had just used the archetype.

### Finishing the day

Well, enough Maven for a day I would say. It came out more verbose than I initially wanted, but without the intro it felt a little lacking. I promise next chapters will be more to the point with more examples and less talking!