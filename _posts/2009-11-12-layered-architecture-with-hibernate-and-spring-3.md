---
title: Layered architecture with Hibernate and Spring 3
author: Carlos Vara
layout: post
permalink: /2009/11/layered-architecture-with-hibernate-and-spring-3/
categories:
  - Java
tags:
  - hibernate
  - Java
  - spring
---
In this post you will learn one of the ways to create a layered data driven application using Hibernate and Spring 3. The architecture will go up from the database to the service layer, so it&#8217;s your choice how to do the presentation part. I will try to adhere to Spring&#8217;s best practices in the separation of layers, so the resulting architecture offers both a clear separation between the layers and little dependencies in the Spring framework.

### Setting up

I use Maven to take care of the compiling and life-cycle of the project. You may use this pom.xml file as the starting point for this project. It basically defines the needed repositories and dependencies that will be used in this guide.

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="xml" style="font-family:monospace;"><span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;project</span> <span style="color: #000066;">xmlns</span>=<span style="color: #ff0000;">"http://maven.apache.org/POM/4.0.0"</span></span>
<span style="color: #009900;">    <span style="color: #000066;">xmlns:xsi</span>=<span style="color: #ff0000;">"http://www.w3.org/2001/XMLSchema-instance"</span></span>
<span style="color: #009900;">    <span style="color: #000066;">xsi:schemaLocation</span>=<span style="color: #ff0000;">"http://maven.apache.org/POM/4.0.0</span>
<span style="color: #009900;">        http://maven.apache.org/maven-v4_0_0.xsd"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>
&nbsp;
	<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;modelVersion<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>4.0.0<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/modelVersion<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
	<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>tld.example<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
	<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>layeredarch-example<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
	<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;packaging<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>war<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/packaging<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
	<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>0.0.1-SNAPSHOT<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
	<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;name<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>Layered Arch Example<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/name<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
&nbsp;
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;properties<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;aspectj.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>1.6.6<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/aspectj.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;commons-dbcp.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>1.2.2<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/commons-dbcp.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;hibernate-annotations.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>3.4.0.GA<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/hibernate-annotations.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;hibernate-core.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>3.3.2.GA<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/hibernate-core.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;hsqldb.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>1.8.0.10<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/hsqldb.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;javassist.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>3.7.ga<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/javassist.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;log4j.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>1.2.15<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/log4j.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;slf4j-log4j12.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>1.5.6<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/slf4j-log4j12.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;springframework.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>3.0.0.RC1<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/springframework.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/properties<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
&nbsp;
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;dependencies<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #808080; font-style: italic;">&lt;!-- Compile time dependencies --&gt;</span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>org.aspectj<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>aspectjrt<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>${aspectj.version}<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>org.aspectj<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>aspectjweaver<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>${aspectj.version}<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>log4j<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>log4j<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>${log4j.version}<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;exclusions<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
               <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;exclusion<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                  <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>javax.jms<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                  <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>jms<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
               <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/exclusion<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
               <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;exclusion<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                  <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>com.sun.jdmk<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                  <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>jmxtools<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
               <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/exclusion<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
               <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;exclusion<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                  <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>com.sun.jmx<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                  <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>jmxri<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
               <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/exclusion<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
               <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;exclusion<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                  <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>javax.mail<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                  <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>mail<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
               <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/exclusion<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/exclusions<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>org.hibernate<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>hibernate-core<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>${hibernate-core.version}<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>org.hibernate<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>hibernate-annotations<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>${hibernate-annotations.version}<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>org.springframework<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>org.springframework.core<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>${springframework.version}<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>org.springframework<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>org.springframework.orm<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>${springframework.version}<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #808080; font-style: italic;">&lt;!-- Runtime dependencies --&gt;</span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>commons-dbcp<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>commons-dbcp<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>${commons-dbcp.version}<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;scope<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>runtime<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/scope<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>hsqldb<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>hsqldb<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>${hsqldb.version}<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;scope<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>runtime<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/scope<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>org.slf4j<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>slf4j-log4j12<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>${slf4j-log4j12.version}<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;scope<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>runtime<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/scope<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>jboss<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>javassist<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>${javassist.version}<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;scope<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>runtime<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/scope<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/dependencies<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
&nbsp;
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;build<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;finalName<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>layeredarch-example<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/finalName<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;plugins<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;plugin<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>org.apache.maven.plugins<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>maven-compiler-plugin<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;configuration<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;source<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>1.6<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/source<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;target<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>1.6<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/target<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/configuration<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/plugin<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/plugins<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/build<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
&nbsp;
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;repositories<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #808080; font-style: italic;">&lt;!-- Legacy java.net repository --&gt;</span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;repository<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;id<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>java-net<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/id<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;url<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>http://download.java.net/maven/1<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/url<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;layout<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>legacy<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/layout<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/repository<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #808080; font-style: italic;">&lt;!-- JBoss repositories: hibernate, etc. --&gt;</span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;repository<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;id<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>jboss<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/id<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;url<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>http://repository.jboss.com/maven2<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/url<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;releases<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;enabled<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>true<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/enabled<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/releases<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;snapshots<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;enabled<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>false<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/enabled<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/snapshots<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/repository<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;repository<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;id<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>jboss-snapshot<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/id<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;url<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>http://snapshots.jboss.org/maven2<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/url<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;releases<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;enabled<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>true<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/enabled<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/releases<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;snapshots<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;enabled<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>true<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/enabled<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/snapshots<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/repository<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #808080; font-style: italic;">&lt;!-- SpringSource repositories --&gt;</span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;repository<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;id<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>springsource-milestone<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/id<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;url<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>http://repository.springsource.com/maven/bundles/milestone<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/url<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/repository<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;repository<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;id<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>springsource-release<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/id<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;url<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>http://repository.springsource.com/maven/bundles/release<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/url<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/repository<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;repository<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;id<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>springsource-external<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/id<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;url<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>http://repository.springsource.com/maven/bundles/external<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/url<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/repository<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/repositories<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
&nbsp;
<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/project<span style="color: #000000; font-weight: bold;">&gt;</span></span></span></pre>
      </td>
    </tr>
  </table>
</div>

### Defining Entities

The Entities represent the domain of your project. They are simple JavaBean classes that also configure how this domain will be persisted to a database. They will be annotated with standard `javax.persistence` annotations so there will be no dependence in neither Hibernate nor Spring.

The following `User.java` is a simple Entity that represents an user in the application. In this example, for every user his name and age are stored along with an auto-generated ID that will identify every persisted user.

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #000000; font-weight: bold;">package</span> <span style="color: #006699;">tld.example.domain</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">javax.persistence.Column</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">javax.persistence.Entity</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">javax.persistence.GeneratedValue</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">javax.persistence.Id</span><span style="color: #339933;">;</span>
&nbsp;
@<span style="color: #003399;">Entity</span>
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> User <span style="color: #009900;">&#123;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">private</span> <span style="color: #003399;">Long</span> id<span style="color: #339933;">;</span>
	<span style="color: #000000; font-weight: bold;">private</span> <span style="color: #003399;">String</span> name<span style="color: #339933;">;</span>
	<span style="color: #000000; font-weight: bold;">private</span> <span style="color: #003399;">Integer</span> age<span style="color: #339933;">;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">public</span> User<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
	@Id
	@GeneratedValue
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">Long</span> getId<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">id</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000066; font-weight: bold;">void</span> setId<span style="color: #009900;">&#40;</span><span style="color: #003399;">Long</span> id<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">id</span> <span style="color: #339933;">=</span> id<span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
	@Column
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">String</span> getName<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">name</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> setName<span style="color: #009900;">&#40;</span><span style="color: #003399;">String</span> name<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">name</span> <span style="color: #339933;">=</span> name<span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
	@Column
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">Integer</span> getAge<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">return</span> age<span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> setAge<span style="color: #009900;">&#40;</span><span style="color: #003399;">Integer</span> age<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">age</span> <span style="color: #339933;">=</span> age<span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>	
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

You may save this class in the tld.example.domain package, where you will also save all the additional Entities that you add.

### DAO Layer

First layer up in the architecture, it&#8217;s the DAO layer. These objects take care of the operations needed to query the database in order to fetch, store and update your Entities. I defined the DAOs in an interface/implementation manner. It&#8217;s not only a good design practice, but it also helps Spring AOP.

The UserDao interface would be as follows:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #000000; font-weight: bold;">package</span> <span style="color: #006699;">tld.example.dao</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">tld.example.domain.User</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">interface</span> UserDao <span style="color: #009900;">&#123;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">public</span> User findById<span style="color: #009900;">&#40;</span><span style="color: #003399;">Long</span> id<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>	
	<span style="color: #000000; font-weight: bold;">public</span> User persistOrMerge<span style="color: #009900;">&#40;</span>User user<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

For the implementation, I have chosen to use Hibernate directly. Nevertheless it&#8217;s quite easy to change it to JPA and the provider you prefer. This is the HibernateUserDao class:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #000000; font-weight: bold;">package</span> <span style="color: #006699;">tld.example.dao.impl</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.hibernate.SessionFactory</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.springframework.beans.factory.annotation.Autowired</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.springframework.stereotype.Repository</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">tld.example.dao.UserDao</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">tld.example.domain.User</span><span style="color: #339933;">;</span>
&nbsp;
@<span style="color: #003399;">Repository</span>
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> HibernateUserDao <span style="color: #000000; font-weight: bold;">implements</span> UserDao <span style="color: #009900;">&#123;</span>
&nbsp;
	@Autowired<span style="color: #009900;">&#40;</span>required<span style="color: #339933;">=</span><span style="color: #000066; font-weight: bold;">true</span><span style="color: #009900;">&#41;</span>
	<span style="color: #000000; font-weight: bold;">private</span> SessionFactory sessionFactory<span style="color: #339933;">;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">public</span> User findById<span style="color: #009900;">&#40;</span><span style="color: #003399;">Long</span> id<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">return</span> <span style="color: #009900;">&#40;</span>User<span style="color: #009900;">&#41;</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">sessionFactory</span>.<span style="color: #006633;">getCurrentSession</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">createQuery</span><span style="color: #009900;">&#40;</span>
			<span style="color: #0000ff;">"from User user where user.id=?"</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">setParameter</span><span style="color: #009900;">&#40;</span><span style="color: #cc66cc;"></span>, id<span style="color: #009900;">&#41;</span>
			.<span style="color: #006633;">uniqueResult</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">public</span> User persistOrMerge<span style="color: #009900;">&#40;</span>User user<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">return</span> <span style="color: #009900;">&#40;</span>User<span style="color: #009900;">&#41;</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">sessionFactory</span>.<span style="color: #006633;">getCurrentSession</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">merge</span><span style="color: #009900;">&#40;</span>user<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

Take into account that there is no transaction management code in this layer. DAOs mission is to abstract the <a href="http://en.wikipedia.org/wiki/Create,_read,_update_and_delete" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://en.wikipedia.org']);">CRUD</a> tasks from your service layer. Transactional logic will be one layer above.

Also, the code exhibits two Spring dependencies because annotations were used to configure the dependency injection of the application. You could easily remove them by moving this configuration to XML.

### Service Layer

This is the highest layer of this example&#8217;s architecture. The service layer provides your application with transactional operations for your business logic. The idea behind this is that a service method is the smallest atomic operation your application will do in the database, so a service method either completes and the resulting database is in consistent status for your application, or rollbacks to its previous state (which should also be consistent).

Again, the services are split into an interface and an implementation. I defined the following simple UserService interface:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #000000; font-weight: bold;">package</span> <span style="color: #006699;">tld.example.service</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">tld.example.domain.User</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">interface</span> UserService <span style="color: #009900;">&#123;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">public</span> User retrieveUser<span style="color: #009900;">&#40;</span><span style="color: #003399;">Long</span> id<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #000000; font-weight: bold;">public</span> User createUser<span style="color: #009900;">&#40;</span>User user<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

And the implementation UserServiceImpl.java:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #000000; font-weight: bold;">package</span> <span style="color: #006699;">tld.example.service.impl</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.springframework.beans.factory.annotation.Autowired</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.springframework.stereotype.Service</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.springframework.transaction.annotation.Transactional</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">tld.example.dao.UserDao</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">tld.example.domain.User</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">tld.example.service.UserService</span><span style="color: #339933;">;</span>
&nbsp;
@Service
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> UserServiceImpl <span style="color: #000000; font-weight: bold;">implements</span> UserService <span style="color: #009900;">&#123;</span>
&nbsp;
	@Autowired<span style="color: #009900;">&#40;</span>required<span style="color: #339933;">=</span><span style="color: #000066; font-weight: bold;">true</span><span style="color: #009900;">&#41;</span>
	<span style="color: #000000; font-weight: bold;">private</span> UserDao userDao<span style="color: #339933;">;</span>
&nbsp;
	@Transactional
	<span style="color: #000000; font-weight: bold;">public</span> User createUser<span style="color: #009900;">&#40;</span>User user<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">userDao</span>.<span style="color: #006633;">persistOrMerge</span><span style="color: #009900;">&#40;</span>user<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
	@Transactional<span style="color: #009900;">&#40;</span>readOnly<span style="color: #339933;">=</span><span style="color: #000066; font-weight: bold;">true</span><span style="color: #009900;">&#41;</span>
	<span style="color: #000000; font-weight: bold;">public</span> User retrieveUser<span style="color: #009900;">&#40;</span><span style="color: #003399;">Long</span> id<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">userDao</span>.<span style="color: #006633;">findById</span><span style="color: #009900;">&#40;</span>id<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

As you can see, when a service method will only perform read operations, you may tell so to Spring and it will be able to optimize the call (this is very useful when the backend is Hibernate).

A very important thing to note here. This is a very simple example with only one Entity and consequently only one DAO, so the Service is very simple. But with this layering, you may very well have Services that use more than one DAO and their functionality spans multiple Entities. The transactional part will take care of that, and you only need to design the Service methods right so they leave the data in the correct status.

### Making it all work together

Now, the last step is to configure Hibernate and Spring to make it all work together. As you will see, thanks to the use of annotations very few config lines are needed.

Hibernate will be mostly managed by Spring, so the only configuration it needs is a pointer to the annotated Entities. In this case, only the User class. This is the hibernate.cfg.xml:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="xml" style="font-family:monospace;"><span style="color: #00bbdd;">&lt;!DOCTYPE hibernate-configuration PUBLIC</span>
<span style="color: #00bbdd;"> "-//Hibernate/Hibernate Configuration DTD 3.0//EN"</span>
<span style="color: #00bbdd;"> "http://hibernate.sourceforge.net/hibernate-configuration-3.0.dtd"&gt;</span>
&nbsp;
<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;hibernate-configuration<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;session-factory<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;mapping</span> <span style="color: #000066;">class</span>=<span style="color: #ff0000;">"tld.example.domain.User"</span> <span style="color: #000000; font-weight: bold;">/&gt;</span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/session-factory<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/hibernate-configuration<span style="color: #000000; font-weight: bold;">&gt;</span></span></span></pre>
      </td>
    </tr>
  </table>
</div>

On the Spring part, a bit more of configuration is needed to set up the <a href="http://static.springsource.org/spring/docs/3.0.x/spring-framework-reference/html/ch10s05.html" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://static.springsource.org']);">declarative transactions</a>. Hsqldb is used for the database. This is the applicationContext.xml file:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="xml" style="font-family:monospace;"><span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;?xml</span> <span style="color: #000066;">version</span>=<span style="color: #ff0000;">"1.0"</span> <span style="color: #000066;">encoding</span>=<span style="color: #ff0000;">"UTF-8"</span><span style="color: #000000; font-weight: bold;">?&gt;</span></span>
<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;beans</span> <span style="color: #000066;">xmlns</span>=<span style="color: #ff0000;">"http://www.springframework.org/schema/beans"</span></span>
<span style="color: #009900;">    <span style="color: #000066;">xmlns:xsi</span>=<span style="color: #ff0000;">"http://www.w3.org/2001/XMLSchema-instance"</span></span>
<span style="color: #009900;">    <span style="color: #000066;">xmlns:aop</span>=<span style="color: #ff0000;">"http://www.springframework.org/schema/aop"</span></span>
<span style="color: #009900;">    <span style="color: #000066;">xmlns:context</span>=<span style="color: #ff0000;">"http://www.springframework.org/schema/context"</span></span>
<span style="color: #009900;">    <span style="color: #000066;">xmlns:tx</span>=<span style="color: #ff0000;">"http://www.springframework.org/schema/tx"</span></span>
<span style="color: #009900;">    <span style="color: #000066;">xsi:schemaLocation</span>=<span style="color: #ff0000;">"</span>
<span style="color: #009900;">     http://www.springframework.org/schema/beans </span>
<span style="color: #009900;">     http://www.springframework.org/schema/beans/spring-beans-3.0.xsd</span>
<span style="color: #009900;">     http://www.springframework.org/schema/context</span>
<span style="color: #009900;">     http://www.springframework.org/schema/context/spring-context-3.0.xsd</span>
<span style="color: #009900;">     http://www.springframework.org/schema/tx</span>
<span style="color: #009900;">     http://www.springframework.org/schema/tx/spring-tx-3.0.xsd</span>
<span style="color: #009900;">     http://www.springframework.org/schema/aop </span>
<span style="color: #009900;">     http://www.springframework.org/schema/aop/spring-aop-3.0.xsd"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>
&nbsp;
    <span style="color: #808080; font-style: italic;">&lt;!-- Configure annotated beans --&gt;</span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;context:annotation-config</span> <span style="color: #000000; font-weight: bold;">/&gt;</span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;context:component-scan</span> <span style="color: #000066;">base-package</span>=<span style="color: #ff0000;">"tld.example"</span> <span style="color: #000000; font-weight: bold;">/&gt;</span></span>
&nbsp;
    <span style="color: #808080; font-style: italic;">&lt;!-- DataSource: hsqldb file --&gt;</span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;bean</span> <span style="color: #000066;">id</span>=<span style="color: #ff0000;">"myDataSource"</span> <span style="color: #000066;">class</span>=<span style="color: #ff0000;">"org.apache.commons.dbcp.BasicDataSource"</span> <span style="color: #000066;">destroy-method</span>=<span style="color: #ff0000;">"close"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;property</span> <span style="color: #000066;">name</span>=<span style="color: #ff0000;">"driverClassName"</span> <span style="color: #000066;">value</span>=<span style="color: #ff0000;">"org.hsqldb.jdbcDriver"</span> <span style="color: #000000; font-weight: bold;">/&gt;</span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;property</span> <span style="color: #000066;">name</span>=<span style="color: #ff0000;">"url"</span> <span style="color: #000066;">value</span>=<span style="color: #ff0000;">"jdbc:hsqldb:file:target/data/example"</span> <span style="color: #000000; font-weight: bold;">/&gt;</span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;property</span> <span style="color: #000066;">name</span>=<span style="color: #ff0000;">"username"</span> <span style="color: #000066;">value</span>=<span style="color: #ff0000;">"sa"</span> <span style="color: #000000; font-weight: bold;">/&gt;</span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;property</span> <span style="color: #000066;">name</span>=<span style="color: #ff0000;">"password"</span> <span style="color: #000066;">value</span>=<span style="color: #ff0000;">""</span> <span style="color: #000000; font-weight: bold;">/&gt;</span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/bean<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
&nbsp;
    <span style="color: #808080; font-style: italic;">&lt;!-- Hibernate --&gt;</span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;bean</span> <span style="color: #000066;">id</span>=<span style="color: #ff0000;">"mySessionFactory"</span> <span style="color: #000066;">class</span>=<span style="color: #ff0000;">"org.springframework.orm.hibernate3.LocalSessionFactoryBean"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;property</span> <span style="color: #000066;">name</span>=<span style="color: #ff0000;">"dataSource"</span> <span style="color: #000066;">ref</span>=<span style="color: #ff0000;">"myDataSource"</span> <span style="color: #000000; font-weight: bold;">/&gt;</span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;property</span> <span style="color: #000066;">name</span>=<span style="color: #ff0000;">"configLocation"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;value<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>classpath:hibernate.cfg.xml<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/value<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/property<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;property</span> <span style="color: #000066;">name</span>=<span style="color: #ff0000;">"configurationClass"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;value<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>org.hibernate.cfg.AnnotationConfiguration<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/value<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/property<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;property</span> <span style="color: #000066;">name</span>=<span style="color: #ff0000;">"hibernateProperties"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;props<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;prop</span> <span style="color: #000066;">key</span>=<span style="color: #ff0000;">"hibernate.show_sql"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>true<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/prop<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;prop</span> <span style="color: #000066;">key</span>=<span style="color: #ff0000;">"hibernate.hbm2ddl.auto"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>create<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/prop<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;prop</span> <span style="color: #000066;">key</span>=<span style="color: #ff0000;">"hibernate.dialect"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>org.hibernate.dialect.HSQLDialect<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/prop<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/props<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/property<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/bean<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
&nbsp;
    <span style="color: #808080; font-style: italic;">&lt;!-- Transaction management --&gt;</span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;tx:annotation-driven</span><span style="color: #000000; font-weight: bold;">/&gt;</span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;bean</span> <span style="color: #000066;">id</span>=<span style="color: #ff0000;">"transactionManager"</span> <span style="color: #000066;">class</span>=<span style="color: #ff0000;">"org.springframework.orm.hibernate3.HibernateTransactionManager"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;property</span> <span style="color: #000066;">name</span>=<span style="color: #ff0000;">"sessionFactory"</span> <span style="color: #000066;">ref</span>=<span style="color: #ff0000;">"mySessionFactory"</span><span style="color: #000000; font-weight: bold;">/&gt;</span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/bean<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
&nbsp;
<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/beans<span style="color: #000000; font-weight: bold;">&gt;</span></span></span></pre>
      </td>
    </tr>
  </table>
</div>

Basically, the following is configured:

  * Spring is told to scan all your classes under the tld.example package and configure the beans according to the annotations. This will create the UserDao and UserService singletons and inject the autowired fields.
  * A DataSource is configured. It uses Apache DBCP for pooling, and hsqldb as Database (both are included in the project dependencies).
  * Spring will inject Hibernate&#8217;s SessionFactory to the DAOs. The `mySessionFactory` bean is all what is needed to do so correctly.
  * And finally, the configuration needed for the declarative transaction management. This code will ensure that all the @Transactional methods in the service layer either run a full transaction or roll-back in case an exception occurs.

And that&#8217;s it <img src="http://carinae.net/wp-includes/images/smilies/icon_biggrin.gif" alt=":D" class="wp-smiley" /> Time for coding all your Entities, DAOs and Services now. There are lot&#8217;s of ways in which you can customize or improve this setup (use JPA, configure Spring&#8217;s exception translator, bean validation, etc.), but the important part is that the layering in the architecture allows to do so easily.