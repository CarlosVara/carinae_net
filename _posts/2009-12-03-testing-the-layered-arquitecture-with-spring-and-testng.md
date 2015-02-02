---
title: Testing the layered arquitecture with Spring and TestNG
author: Carlos Vara
layout: post
permalink: /2009/12/testing-the-layered-arquitecture-with-spring-and-testng/
categories:
  - Java
tags:
  - spring
  - testng
---
Following the post explaining a <a href="http://carinae.net/2009/11/layered-architecture-with-hibernate-and-spring-3/" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://carinae.net']);">layered arquitecture with Spring and Hibernate</a>, this entry will explain how to easily test its DAOs and Service components using Spring&#8217;s TestNG integration.

When it comes to isolating the environment for each layer, the main conceptual difference between the layers is this: the service layer is transactional on its own, so its methods can be tested without adding any extra components; but the DAO layer requires an ongoing transaction for its methods to work, so one must be supplied.

### Preparing the environment

Two extra dependencies must be added in the project&#8217;s `pom.xml` file: spring test context framework, which has the helper classes for different test libraries, and the <a href="http://testng.org" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://testng.org']);">TestNG</a> library, which is the framework that is going to be used. Both will be added with test scope, as they only have to be present in the classpath during that phase.

This is the relevant fragment of the `pom.xml` file:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="xml" style="font-family:monospace;"><span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;dependencies<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        [...]
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>org.testng<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>testng<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>${testng.version}<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;classifier<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>jdk15<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/classifier<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;scope<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>test<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/scope<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>org.springframework<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/groupId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>org.springframework.test<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/artifactId<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>${springframework.version}<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;scope<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>test<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/scope<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/dependency<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
	[...]
<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/dependencies<span style="color: #000000; font-weight: bold;">&gt;</span></span></span></pre>
      </td>
    </tr>
  </table>
</div>

Take care that you will need to define the properties holding the version values for the springframework and testng dependencies if you don&#8217;t have them already. In this entry, I assume the following versions:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="xml" style="font-family:monospace;"><span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;properties<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
	[...]
	<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;springframework.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>3.0.0.RC2<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/springframework.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
	<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;testng.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>5.10<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/testng.version<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
	[...]
<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/properties<span style="color: #000000; font-weight: bold;">&gt;</span></span></span></pre>
      </td>
    </tr>
  </table>
</div>

### Testing the DAO layer

So, DAO methods should be executed inside a transactional scope. To provide it, the test class will inherit from `AbstractTransactionalTestNGSpringContextTests`. As an extra, you may provide it with different spring application context configuration to adapt it to your test. Just be sure that in case you use a different configuration, it includes a TransactionManager so it can be used to create the transactions.

An example of a DAO test would be like this:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;">@ContextConfiguration<span style="color: #009900;">&#40;</span>locations <span style="color: #339933;">=</span> <span style="color: #009900;">&#123;</span> <span style="color: #0000ff;">"classpath:applicationContext.xml"</span> <span style="color: #009900;">&#125;</span><span style="color: #009900;">&#41;</span>
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> UserDaoTest 
		<span style="color: #000000; font-weight: bold;">extends</span> AbstractTransactionalTestNGSpringContextTests <span style="color: #009900;">&#123;</span>
&nbsp;
	@Autowired
	<span style="color: #000000; font-weight: bold;">private</span> UserDao userDao<span style="color: #339933;">;</span>
&nbsp;
	@Test
	@Rollback<span style="color: #009900;">&#40;</span><span style="color: #000066; font-weight: bold;">true</span><span style="color: #009900;">&#41;</span>
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> simpleTest<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		User user1 <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">userDao</span>.<span style="color: #006633;">findById</span><span style="color: #009900;">&#40;</span>1l<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
		assertNotNull<span style="color: #009900;">&#40;</span>user1, <span style="color: #0000ff;">"User 1 could not be retrieved."</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

The rollback annotation allows you to decide whether the supplied transaction should proceed or be rolled back at the end of the test. In this case it doesn&#8217;t matter as the test is of a read-only operation, but it&#8217;s very handy when testing write operations.

### Testing the service layer

Testing the service layer is even easier as it doesn&#8217;t need any extra scope or configuration to work. Still, to get the extra value provided by the Spring Test Framework (selection of the spring configuration, context caching, dependency injection, etc.) it is a good idea to inherit from the `AbstractTestNGSpringContextTests` class.

An example of a service test would look like this:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;">@ContextConfiguration<span style="color: #009900;">&#40;</span> locations<span style="color: #339933;">=</span><span style="color: #009900;">&#123;</span><span style="color: #0000ff;">"classpath:applicationContext.xml"</span><span style="color: #009900;">&#125;</span> <span style="color: #009900;">&#41;</span>
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> UserServiceTest <span style="color: #000000; font-weight: bold;">extends</span> AbstractTestNGSpringContextTests <span style="color: #009900;">&#123;</span>
&nbsp;
	@Autowired
	<span style="color: #000000; font-weight: bold;">private</span> UserService userService<span style="color: #339933;">;</span>
&nbsp;
	@Test
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> simpleTest<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		Collection<span style="color: #339933;">&lt;</span>User<span style="color: #339933;">&gt;</span> users <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">userService</span>.<span style="color: #006633;">getAllUsers</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
		assertEquals<span style="color: #009900;">&#40;</span>users.<span style="color: #006633;">size</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>, <span style="color: #cc66cc;">3</span>,
			<span style="color: #0000ff;">"Incorrect number of users retrieved."</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

Take note that to unit test the service layer, you should provide a mock DAO to the service so its operations are tested in isolation. Spring&#8217;s dependency injection comes again handy in this regard: as the DAO is injected into the service, it&#8217;s easy to adapt the test configuration so the service gets a mocked DAO instead. How to configure that may be explained in a following post.