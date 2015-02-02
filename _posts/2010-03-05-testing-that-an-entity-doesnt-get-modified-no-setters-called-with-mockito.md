---
title: 'Testing that an entity doesn&#8217;t get modified (no setters called) with mockito'
author: Carlos Vara
layout: post
permalink: /2010/03/testing-that-an-entity-doesnt-get-modified-no-setters-called-with-mockito/
categories:
  - Java
tags:
  - jpa
  - mockito
  - testing
---
I have recently started using <a href="http://mockito.org/" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://mockito.org']);">mockito</a> to improve the quality of my unit tests. It&#8217;s a great tool, with a very clear syntax that makes tests very readable. Also, if you follow a <a href="http://en.wikipedia.org/wiki/Behavior_Driven_Development" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://en.wikipedia.org']);">BDD</a> pattern in your tests, mockito has aliases in BDDMockito so that your actions can clearly follow the <a href="http://monkeyisland.pl/2009/12/07/given-when-then-forever/" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://monkeyisland.pl']);">given/then/when</a> template.

I&#8217;m using it to test the behaviour of some transactional services. The underlying persistence technology used is JPA, so checking that some entity is not modified is a bit hard. Why? Because with JPA, the usual way of updating an entity is by bringing it into your persistence context, modifying it, and then simply closing the context. The entity manager will detect that the retrieved entity has changed, and will create the appropriate update statement.

So, if I want to ensure that a method in the service layer doesn&#8217;t modify an entity, a clean way of doing it is ensuring that no setter methods are called on that entity. In order to be able to test this with mockito, you need to do two things: first, make the mocked DAO return a &#8220;spy&#8221; of the entity; and second, verify that no spurious set* methods have been called on that spy during the service call.

The fist part is the easy one, simply create the entity that should be returned, and once it has all of its expected values set, wrap it in a spy.

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;">MyEntity entity <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">new</span> MyEntity<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
entity.<span style="color: #006633;">setId</span><span style="color: #009900;">&#40;</span><span style="color: #cc66cc;">1</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
entity.<span style="color: #006633;">setText</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"hello"</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
entity <span style="color: #339933;">=</span> spy<span style="color: #009900;">&#40;</span>entity<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
given<span style="color: #009900;">&#40;</span>mockedEntityDao.<span style="color: #006633;">findEntityById</span><span style="color: #009900;">&#40;</span><span style="color: #cc66cc;">1</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">willReturn</span><span style="color: #009900;">&#40;</span>entity<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span></pre>
      </td>
    </tr>
  </table>
</div>

The second part is trickier. A new verification mode needs to be created. Its verify method will have access to the list of all the called methods in the spy, and it&#8217;s simply a matter of checking that no method that has the signature of a setter (its name matches the expression set[A-Z].* and has 1 parameter) has been called. The code of this verification mode is as follows:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">java.util.List</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">java.util.regex.Matcher</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">java.util.regex.Pattern</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.mockito.exceptions.Reporter</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.mockito.internal.invocation.Invocation</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.mockito.internal.verification.api.VerificationData</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.mockito.internal.verification.api.VerificationMode</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #008000; font-style: italic; font-weight: bold;">/**
 * Mockito verification mode to ensure that no setter calls have been performed
 * on a mock/spy.
 * 
 * @author Carlos Vara
 */</span>
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> NoSetterCalls <span style="color: #000000; font-weight: bold;">implements</span> VerificationMode <span style="color: #009900;">&#123;</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000000; font-weight: bold;">final</span> Reporter reporter<span style="color: #339933;">;</span>
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000000; font-weight: bold;">final</span> Pattern regexPattern<span style="color: #339933;">;</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">public</span> NoSetterCalls<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">reporter</span> <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">new</span> Reporter<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">regexPattern</span> <span style="color: #339933;">=</span> Pattern.<span style="color: #006633;">compile</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"set[A-Z].*"</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    @Override
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> verify<span style="color: #009900;">&#40;</span>VerificationData data<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
        List<span style="color: #339933;">&lt;</span>Invocation<span style="color: #339933;">&gt;</span> invocations <span style="color: #339933;">=</span> data.<span style="color: #006633;">getAllInvocations</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #000000; font-weight: bold;">for</span> <span style="color: #009900;">&#40;</span> Invocation inv <span style="color: #339933;">:</span> invocations <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
            Matcher m <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">regexPattern</span>.<span style="color: #006633;">matcher</span><span style="color: #009900;">&#40;</span>inv.<span style="color: #006633;">getMethodName</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
            <span style="color: #000000; font-weight: bold;">if</span> <span style="color: #009900;">&#40;</span> m.<span style="color: #006633;">matches</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #339933;">&&</span> inv.<span style="color: #006633;">getArgumentsCount</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #339933;">==</span> <span style="color: #cc66cc;">1</span> <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
                <span style="color: #666666; font-style: italic;">// A setter has been called!</span>
                <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">reporter</span>.<span style="color: #006633;">neverWantedButInvoked</span><span style="color: #009900;">&#40;</span>inv, inv.<span style="color: #006633;">getLocation</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
            <span style="color: #009900;">&#125;</span>
&nbsp;
        <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #009900;">&#125;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

And finally, an example test would be as follows:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;">@Test
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> checkSimpleAlternativeParagraph<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
    <span style="color: #666666; font-style: italic;">// GIVEN</span>
    <span style="color: #666666; font-style: italic;">//  - A valid call to a service methodproposal to create a new alternative</span>
    <span style="color: #003399;">String</span> callParam <span style="color: #339933;">=</span> <span style="color: #0000ff;">"1"</span><span style="color: #339933;">;</span>
    <span style="color: #666666; font-style: italic;">//  - And the expected DAO behaviour</span>
    MyEntity entity <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">new</span> MyEntity<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    entity.<span style="color: #006633;">setId</span><span style="color: #009900;">&#40;</span><span style="color: #cc66cc;">1</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    entity.<span style="color: #006633;">setText</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"hello"</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    entity <span style="color: #339933;">=</span> spy<span style="color: #009900;">&#40;</span>entity<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    given<span style="color: #009900;">&#40;</span>mockedEntityDao.<span style="color: #006633;">findEntityById</span><span style="color: #009900;">&#40;</span><span style="color: #cc66cc;">1</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">willReturn</span><span style="color: #009900;">&#40;</span>entity<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
    <span style="color: #666666; font-style: italic;">// WHEN</span>
    <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">entityService</span>.<span style="color: #006633;">doSomethingButDontModify</span><span style="color: #009900;">&#40;</span>callParam<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
    <span style="color: #666666; font-style: italic;">// THEN</span>
    <span style="color: #666666; font-style: italic;">//  - Verify that the queried entity hasn't been modified</span>
    verify<span style="color: #009900;">&#40;</span>entity, <span style="color: #000000; font-weight: bold;">new</span> NoSetterCalls<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">setId</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"this setId() call is ignored"</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

And it works. The only drawback, is that you have to complete the verification call with an extra method call, the setId() call in this case, but could be any other method. This call has no side-effects (it gets passed as parameter to the NoSetterCalls verifier) but disturbs a little the clarity of the test.

### Note when using JPA2

In JPA2, the entity manager has a detach method. If your application uses it, the way to test that no modification is performed would be to check that no setters followed by a flush are performed in the entity prior to calling detach. Or more easily, add a DAO method that returns a detached entity, and move this check to the DAO.