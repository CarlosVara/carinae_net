---
title: Automatic validation of method calls with JSR-303 (Appendix-C of the specification)
author: Carlos Vara
layout: post
permalink: /2010/04/automatic-validation-method-calls-with-jsr-303-appendix-c/
categories:
  - Java
tags:
  - apache
  - aspectj
  - beanvalidation
  - jsr-303
  - validation
---
The recently approved Bean Validation Standard (JSR-303) left one great (and requested) feature out of the specification: method validation. This proposal defined an additional API for the Validator with methods that allowed validation of method/constructor parameters as well as the return value of methods. Thankfully, even though this spec didn&#8217;t make it to the final approved document, all constraint annotations accept `parameter` as target, so the door was left open for it to be implemented as an extra feature.  
<a href="http://carinae.net/wp-content/uploads/2010/04/methodvalidation.tar.gz" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://carinae.net']);">methodvalidation.tar</a>

### Apache BeanValidation

There are currently 2 implementations of the standard, <a href="http://www.hibernate.org/subprojects/validator.html" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://www.hibernate.org']);">Hibernate Validator</a> and <a href="http://incubator.apache.org/projects/beanvalidation.html" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://incubator.apache.org']);">Apache BeanValidation</a> (formerly agimatec-validator, and as of this march an Apache Incubator project). Of these two, only Apache BeanValidation supports the additional method validation API, so it is the only choice if you need that feature, and it&#8217;s what I will use as base for this example.

### Method validation API

The proposed additional methods in the bean validation are the following:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;</span> Set<span style="color: #339933;">&lt;</span>ConstraintViolation<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;&gt;</span> validateParameters<span style="color: #009900;">&#40;</span>Class<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;</span> clazz, <span style="color: #003399;">Method</span> method,
                                                       <span style="color: #003399;">Object</span><span style="color: #009900;">&#91;</span><span style="color: #009900;">&#93;</span> parameterValues,
                                                       Class<span style="color: #339933;">&lt;?&gt;</span>... <span style="color: #006633;">groups</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;</span> Set<span style="color: #339933;">&lt;</span>ConstraintViolation<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;&gt;</span> validateParameter<span style="color: #009900;">&#40;</span>Class<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;</span> clazz, <span style="color: #003399;">Method</span> method,
                                                   <span style="color: #003399;">Object</span> parameterValue,
                                                   <span style="color: #000066; font-weight: bold;">int</span> parameterIndex,
                                                   Class<span style="color: #339933;">&lt;?&gt;</span>... <span style="color: #006633;">groups</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;</span> Set<span style="color: #339933;">&lt;</span>ConstraintViolation<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;&gt;</span> validateReturnedValue<span style="color: #009900;">&#40;</span>Class<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;</span> clazz, <span style="color: #003399;">Method</span> method,
                                                       <span style="color: #003399;">Object</span> returnedValue,
                                                       Class<span style="color: #339933;">&lt;?&gt;</span>... <span style="color: #006633;">groups</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;</span> Set<span style="color: #339933;">&lt;</span>ConstraintViolation<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;&gt;</span> validateParameters<span style="color: #009900;">&#40;</span>Class<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;</span> clazz,
                                                    <span style="color: #003399;">Constructor</span> constructor,
                                                    <span style="color: #003399;">Object</span><span style="color: #009900;">&#91;</span><span style="color: #009900;">&#93;</span> parameterValues,
                                                    Class<span style="color: #339933;">&lt;?&gt;</span>... <span style="color: #006633;">groups</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
&nbsp;
<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;</span> Set<span style="color: #339933;">&lt;</span>ConstraintViolation<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;&gt;</span> validateParameter<span style="color: #009900;">&#40;</span>Class<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;</span> clazz,
                                                   <span style="color: #003399;">Constructor</span> constructor,
                                                   <span style="color: #003399;">Object</span> parameterValue,
                                                   <span style="color: #000066; font-weight: bold;">int</span> parameterIndex,
                                                   Class<span style="color: #339933;">&lt;?&gt;</span>... <span style="color: #006633;">groups</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span></pre>
      </td>
    </tr>
  </table>
</div>

So, to validate the parameters of a method call, one would call `validateParameters` with the holder class, the method description and the parameter values as parameters, and the output would be similar than when validating a bean.

And how do you specify the constraints? In the method declaration, as in this example:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;">@NotNull
@<span style="color: #003399;">NotEmpty</span>
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">String</span> operation<span style="color: #009900;">&#40;</span>@NotNull @Pattern<span style="color: #009900;">&#40;</span>regexp<span style="color: #339933;">=</span><span style="color: #0000ff;">"[0-9]{2}"</span><span style="color: #009900;">&#41;</span> <span style="color: #003399;">String</span> param<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
   <span style="color: #666666; font-style: italic;">// Your code</span>
   <span style="color: #000000; font-weight: bold;">return</span> val<span style="color: #339933;">;</span>
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

This enhanced method declaration indicates that the `param` value cannot be null, and must be matched by the regular expression `[0-9]{2}`. In the same way, the value returned by the function cannot be null or an empty string.

### Automatic validation using AspectJ

Being one good example of a <a href="http://en.wikipedia.org/wiki/Cross-cutting_concern" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://en.wikipedia.org']);">crosscutting concern</a>, validation code can easily pollute all your application code and maintenance can become really difficult. So, a good way to implement it automatically is using <a href="http://eclipse.org/aspectj/" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://eclipse.org']);">AspectJ</a>. This way, you will decide in a single place (the pointcut) what method and constructors you want to be validated, and the validation code will also be centralized in a single place (the advice).

The aspect implementing this functionality is as follows:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #000000; font-weight: bold;">package</span> <span style="color: #006699;">net.carinae.methodvalidation</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">java.util.Arrays</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">java.util.Set</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">javax.validation.ConstraintViolation</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">javax.validation.Validation</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">javax.validation.ValidationException</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">javax.validation.ValidatorFactory</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.apache.bval.jsr303.extensions.MethodValidator</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.aspectj.lang.reflect.ConstructorSignature</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.aspectj.lang.reflect.MethodSignature</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.slf4j.Logger</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.slf4j.LoggerFactory</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #008000; font-style: italic; font-weight: bold;">/**
 * Enforces correct parameters and return values on the adviced methods and constructors.
 * &lt;p&gt;
 * NOTE: Currently only works with Apache BeanValidation.
 * 
 * @author Carlos Vara
 */</span>
<span style="color: #000000; font-weight: bold;">public</span> aspect MethodValidationAspect <span style="color: #009900;">&#123;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">final</span> <span style="color: #000000; font-weight: bold;">static</span> Logger logger <span style="color: #339933;">=</span> LoggerFactory.<span style="color: #006633;">getLogger</span><span style="color: #009900;">&#40;</span>MethodValidationAspect.<span style="color: #000000; font-weight: bold;">class</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">static</span> <span style="color: #000000; font-weight: bold;">private</span> ValidatorFactory factory<span style="color: #339933;">;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">static</span> <span style="color: #009900;">&#123;</span>
		factory <span style="color: #339933;">=</span> Validation.<span style="color: #006633;">buildDefaultValidatorFactory</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">static</span> <span style="color: #000000; font-weight: bold;">private</span> MethodValidator getMethodValidator<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">return</span> factory.<span style="color: #006633;">getValidator</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">unwrap</span><span style="color: #009900;">&#40;</span>MethodValidator.<span style="color: #000000; font-weight: bold;">class</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
	pointcut validatedMethodCall<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #339933;">:</span> execution<span style="color: #009900;">&#40;</span>@ValidatedMethodCall <span style="color: #339933;">*</span> <span style="color: #339933;">*</span><span style="color: #009900;">&#40;</span>..<span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
	pointcut validatedConstructorCall<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #339933;">:</span> execution<span style="color: #009900;">&#40;</span>@ValidatedConstructorCall <span style="color: #339933;">*</span> .<span style="color: #000000; font-weight: bold;">new</span><span style="color: #009900;">&#40;</span>..<span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
	pointcut validatedReturnValue<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #339933;">:</span> validatedMethodCall<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #339933;">&&</span> execution<span style="color: #009900;">&#40;</span><span style="color: #339933;">!</span><span style="color: #000066; font-weight: bold;">void</span> <span style="color: #339933;">*</span><span style="color: #009900;">&#40;</span>..<span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
&nbsp;
	<span style="color: #008000; font-style: italic; font-weight: bold;">/**
	 * Validates the method parameters.
	 */</span>
	before<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #339933;">:</span> validatedMethodCall<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
		MethodSignature methodSignature <span style="color: #339933;">=</span> <span style="color: #009900;">&#40;</span>MethodSignature<span style="color: #009900;">&#41;</span>thisJoinPoint.<span style="color: #006633;">getSignature</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
		logger.<span style="color: #006633;">trace</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Validating call: {} with args {}"</span>, methodSignature.<span style="color: #006633;">getMethod</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>, <span style="color: #003399;">Arrays</span>.<span style="color: #006633;">toString</span><span style="color: #009900;">&#40;</span>thisJoinPoint.<span style="color: #006633;">getArgs</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
		Set<span style="color: #339933;">&lt;?</span> <span style="color: #000000; font-weight: bold;">extends</span> ConstraintViolation<span style="color: #339933;">&lt;?&gt;&gt;</span> validationErrors <span style="color: #339933;">=</span> getMethodValidator<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">validateParameters</span><span style="color: #009900;">&#40;</span>thisJoinPoint.<span style="color: #006633;">getThis</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">getClass</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>, methodSignature.<span style="color: #006633;">getMethod</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>, thisJoinPoint.<span style="color: #006633;">getArgs</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
		<span style="color: #000000; font-weight: bold;">if</span> <span style="color: #009900;">&#40;</span> validationErrors.<span style="color: #006633;">isEmpty</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
			logger.<span style="color: #006633;">trace</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Valid call"</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
		<span style="color: #009900;">&#125;</span>
		<span style="color: #000000; font-weight: bold;">else</span> <span style="color: #009900;">&#123;</span>
			logger.<span style="color: #006633;">warn</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Invalid call"</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
			<span style="color: #003399;">RuntimeException</span> ex <span style="color: #339933;">=</span> buildValidationException<span style="color: #009900;">&#40;</span>validationErrors<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
			<span style="color: #000000; font-weight: bold;">throw</span> ex<span style="color: #339933;">;</span>
		<span style="color: #009900;">&#125;</span>
&nbsp;
	<span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
	<span style="color: #008000; font-style: italic; font-weight: bold;">/**
	 * Validates the constructor parameters.
	 */</span>
	before<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #339933;">:</span> validatedConstructorCall<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
		ConstructorSignature constructorSignature <span style="color: #339933;">=</span> <span style="color: #009900;">&#40;</span>ConstructorSignature<span style="color: #009900;">&#41;</span>thisJoinPoint.<span style="color: #006633;">getSignature</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
		logger.<span style="color: #006633;">trace</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Validating constructor: {} with args {}"</span>, constructorSignature.<span style="color: #006633;">getConstructor</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>, <span style="color: #003399;">Arrays</span>.<span style="color: #006633;">toString</span><span style="color: #009900;">&#40;</span>thisJoinPoint.<span style="color: #006633;">getArgs</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
		Set<span style="color: #339933;">&lt;?</span> <span style="color: #000000; font-weight: bold;">extends</span> ConstraintViolation<span style="color: #339933;">&lt;?&gt;&gt;</span> validationErrors <span style="color: #339933;">=</span> getMethodValidator<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">validateParameters</span><span style="color: #009900;">&#40;</span>thisJoinPoint.<span style="color: #006633;">getThis</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">getClass</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>, constructorSignature.<span style="color: #006633;">getConstructor</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>, thisJoinPoint.<span style="color: #006633;">getArgs</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
		<span style="color: #000000; font-weight: bold;">if</span> <span style="color: #009900;">&#40;</span> validationErrors.<span style="color: #006633;">isEmpty</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
			logger.<span style="color: #006633;">trace</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Valid call"</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
		<span style="color: #009900;">&#125;</span>
		<span style="color: #000000; font-weight: bold;">else</span> <span style="color: #009900;">&#123;</span>
			logger.<span style="color: #006633;">warn</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Invalid call"</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
			<span style="color: #003399;">RuntimeException</span> ex <span style="color: #339933;">=</span> buildValidationException<span style="color: #009900;">&#40;</span>validationErrors<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
			<span style="color: #000000; font-weight: bold;">throw</span> ex<span style="color: #339933;">;</span>
		<span style="color: #009900;">&#125;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
	<span style="color: #008000; font-style: italic; font-weight: bold;">/**
	 * Validates the returned value of a method call.
	 * 
	 * @param ret The returned value
	 */</span>
	after<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> returning<span style="color: #009900;">&#40;</span><span style="color: #003399;">Object</span> ret<span style="color: #009900;">&#41;</span> <span style="color: #339933;">:</span> validatedReturnValue<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
		MethodSignature methodSignature <span style="color: #339933;">=</span> <span style="color: #009900;">&#40;</span>MethodSignature<span style="color: #009900;">&#41;</span>thisJoinPoint.<span style="color: #006633;">getSignature</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
		logger.<span style="color: #006633;">trace</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Validating returned value {} from call: {}"</span>, ret, methodSignature.<span style="color: #006633;">getMethod</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
		Set<span style="color: #339933;">&lt;?</span> <span style="color: #000000; font-weight: bold;">extends</span> ConstraintViolation<span style="color: #339933;">&lt;?&gt;&gt;</span> validationErrors <span style="color: #339933;">=</span> getMethodValidator<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">validateReturnedValue</span><span style="color: #009900;">&#40;</span>thisJoinPoint.<span style="color: #006633;">getThis</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">getClass</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>, methodSignature.<span style="color: #006633;">getMethod</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>, ret<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
		<span style="color: #000000; font-weight: bold;">if</span> <span style="color: #009900;">&#40;</span> validationErrors.<span style="color: #006633;">isEmpty</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
			logger.<span style="color: #006633;">info</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Valid call"</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
		<span style="color: #009900;">&#125;</span>
		<span style="color: #000000; font-weight: bold;">else</span> <span style="color: #009900;">&#123;</span>
			logger.<span style="color: #006633;">warn</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Invalid call"</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
			<span style="color: #003399;">RuntimeException</span> ex <span style="color: #339933;">=</span> buildValidationException<span style="color: #009900;">&#40;</span>validationErrors<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
			<span style="color: #000000; font-weight: bold;">throw</span> ex<span style="color: #339933;">;</span>
		<span style="color: #009900;">&#125;</span>
&nbsp;
	<span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
	<span style="color: #008000; font-style: italic; font-weight: bold;">/**
	 * @param validationErrors The errors detected in a method/constructor call.
	 * @return A RuntimeException with information about the detected validation errors. 
	 */</span>
	<span style="color: #000000; font-weight: bold;">private</span> <span style="color: #003399;">RuntimeException</span> buildValidationException<span style="color: #009900;">&#40;</span>Set<span style="color: #339933;">&lt;?</span> <span style="color: #000000; font-weight: bold;">extends</span> ConstraintViolation<span style="color: #339933;">&lt;?&gt;&gt;</span> validationErrors<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		StringBuilder sb <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">new</span> StringBuilder<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
		<span style="color: #000000; font-weight: bold;">for</span> <span style="color: #009900;">&#40;</span>ConstraintViolation<span style="color: #339933;">&lt;?&gt;</span> cv <span style="color: #339933;">:</span> validationErrors <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
			sb.<span style="color: #006633;">append</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"<span style="color: #000099; font-weight: bold;">\n</span>"</span> <span style="color: #339933;">+</span> cv.<span style="color: #006633;">getPropertyPath</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #339933;">+</span> <span style="color: #0000ff;">"{"</span> <span style="color: #339933;">+</span> cv.<span style="color: #006633;">getInvalidValue</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #339933;">+</span> <span style="color: #0000ff;">"} : "</span> <span style="color: #339933;">+</span> cv.<span style="color: #006633;">getMessage</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
		<span style="color: #009900;">&#125;</span>
		<span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">new</span> ValidationException<span style="color: #009900;">&#40;</span>sb.<span style="color: #006633;">toString</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

I have defined 3 simple pointcuts (to advice parameter validation of method and constructors, and return value of methods), and I have used 2 custom made interfaces to annotate the methods in which I want validation to be performed. You may of course want to tweak those pointcuts to adapt to your environment. The code of the 2 interfaces is included in the packaged project.

### Some gotchas

Take care that the current implementation of the method validation API is still experimental. As of this writing, many constraints still don&#8217;t work (I used a patched build for the @Size and @Pattern constraints to work), but it&#8217;s just a matter of time that all the features available for bean validation work as well for parameter validation.

If you want to start from a template, I have attached a simple maven project that shows the use of this technique and has the aspect and needed interfaces code in it.  
Download it here: <a href="http://carinae.net/wp-content/uploads/2010/04/methodvalidation.tar.gz" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://carinae.net']);">methodvalidation.tar.gz</a>