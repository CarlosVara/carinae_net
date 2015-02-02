---
title: Integration of JSR 303 bean validation standard and Wicket 1.4
author: Carlos Vara
layout: post
permalink: /2009/12/integration-of-jsr-303-bean-validation-standard-and-wicket-1-4/
categories:
  - Java
tags:
  - hibernate-validator
  - i18n
  - jsr-303
  - spring
  - validation
  - wicket
---
In this entry I will show a way to integrate the new <a href="http://jcp.org/en/jsr/detail?id=303" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://jcp.org']);">JSR 303 bean validation standard</a> with <a href="http://wicket.apache.org" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://wicket.apache.org']);">Wicket</a> 1.4. The resulting example form will have AJAX callbacks to inform the user promptly about validation errors and these messages will be internationalized according to the locale associated with the user&#8217;s session. Spring 3 will be used to manage the Validator instance.

To get in perspective, in this example a user registration form (inside of a panel) will be created. The form will have 4 inputs: email, password, password verification and user age. When a user fills an input, an AJAX callback to the server will be done to validate that input. In case of a validation error, the reported errors will appear next to that input.

### The UserRegistrationPanel

This panel will contain two components: the form and a feedback panel where the errors which are not associated to a single input will be reported (when the 2 supplied passwords don&#8217;t match for example).

The panel markup is as follows:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="xml" style="font-family:monospace;"><span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;html</span> <span style="color: #000066;">xmlns:wicket</span>=<span style="color: #ff0000;">"http://wicket.apache.org/dtds.data/wicket-xhtml1.3-strict.dtd"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>         
<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;body<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;wicket:panel<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
&nbsp;
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;form</span> <span style="color: #000066;">wicket:id</span>=<span style="color: #ff0000;">"registrationForm"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;fieldset<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>        
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;div</span> <span style="color: #000066;">wicket:id</span>=<span style="color: #ff0000;">"validatedEmailBorder"</span> <span style="color: #000066;">class</span>=<span style="color: #ff0000;">"entry"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;label</span> <span style="color: #000066;">for</span>=<span style="color: #ff0000;">"email"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>Your e-mail:<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/label<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;input</span> <span style="color: #000066;">wicket:id</span>=<span style="color: #ff0000;">"email"</span> <span style="color: #000066;">name</span>=<span style="color: #ff0000;">"email"</span> <span style="color: #000066;">type</span>=<span style="color: #ff0000;">"text"</span> <span style="color: #000000; font-weight: bold;">/&gt;</span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/div<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>            
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;div</span> <span style="color: #000066;">wicket:id</span>=<span style="color: #ff0000;">"validatedPasswordBorder"</span> <span style="color: #000066;">class</span>=<span style="color: #ff0000;">"entry"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;label</span> <span style="color: #000066;">for</span>=<span style="color: #ff0000;">"password"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>Choose password:<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/label<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;input</span> <span style="color: #000066;">wicket:id</span>=<span style="color: #ff0000;">"password"</span> <span style="color: #000066;">name</span>=<span style="color: #ff0000;">"password"</span> <span style="color: #000066;">type</span>=<span style="color: #ff0000;">"password"</span> <span style="color: #000000; font-weight: bold;">/&gt;</span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/div<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>            
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;div</span> <span style="color: #000066;">wicket:id</span>=<span style="color: #ff0000;">"validatedPasswordVerificationBorder"</span> <span style="color: #000066;">class</span>=<span style="color: #ff0000;">"entry"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;label</span> <span style="color: #000066;">for</span>=<span style="color: #ff0000;">"passwordVerification"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>Re-type password:<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/label<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;input</span> <span style="color: #000066;">wicket:id</span>=<span style="color: #ff0000;">"passwordVerification"</span> <span style="color: #000066;">name</span>=<span style="color: #ff0000;">"passwordVerification"</span> <span style="color: #000066;">type</span>=<span style="color: #ff0000;">"password"</span> <span style="color: #000000; font-weight: bold;">/&gt;</span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/div<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>            
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;div</span> <span style="color: #000066;">wicket:id</span>=<span style="color: #ff0000;">"validatedAgeBorder"</span> <span style="color: #000066;">class</span>=<span style="color: #ff0000;">"entry"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;label</span> <span style="color: #000066;">for</span>=<span style="color: #ff0000;">"age"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>Your age:<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/label<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
                <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;input</span> <span style="color: #000066;">wicket:id</span>=<span style="color: #ff0000;">"age"</span> <span style="color: #000066;">name</span>=<span style="color: #ff0000;">"age"</span> <span style="color: #000066;">type</span>=<span style="color: #ff0000;">"text"</span> <span style="color: #000000; font-weight: bold;">/&gt;</span></span>
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/div<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>        
            <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;input</span> <span style="color: #000066;">type</span>=<span style="color: #ff0000;">"submit"</span> <span style="color: #000066;">value</span>=<span style="color: #ff0000;">"Register!"</span><span style="color: #000000; font-weight: bold;">/&gt;</span></span>
        <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/fieldset<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/form<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
&nbsp;
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;div</span> <span style="color: #000066;">wicket:id</span>=<span style="color: #ff0000;">"feedback"</span> <span style="color: #000066;">class</span>=<span style="color: #ff0000;">"feedback"</span><span style="color: #000000; font-weight: bold;">&gt;</span><span style="color: #000000; font-weight: bold;">&lt;/div<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
&nbsp;
<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/wicket:panel<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/body<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/html<span style="color: #000000; font-weight: bold;">&gt;</span></span></span></pre>
      </td>
    </tr>
  </table>
</div>

Only one thing to explain in here, the inputs are surrounded with a border component. This way, it will be easy to control the extra markup needed to show the input related validation errors.

And now, the associated code `UserRegistrationPanel.java`:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> UserRegistrationPanel <span style="color: #000000; font-weight: bold;">extends</span> <span style="color: #003399;">Panel</span> <span style="color: #009900;">&#123;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">public</span> UserRegistrationPanel<span style="color: #009900;">&#40;</span><span style="color: #003399;">String</span> id<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">super</span><span style="color: #009900;">&#40;</span>id<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
		<span style="color: #666666; font-style: italic;">// Insert the form and the feedback div</span>
		RegistrationForm regForm<span style="color: #339933;">;</span>
		add<span style="color: #009900;">&#40;</span>regForm <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">new</span> RegistrationForm<span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"registrationForm"</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
		add<span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">new</span> FeedbackPanel<span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"feedback"</span>, <span style="color: #000000; font-weight: bold;">new</span> ComponentFeedbackMessageFilter<span style="color: #009900;">&#40;</span>regForm<span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">setOutputMarkupId</span><span style="color: #009900;">&#40;</span><span style="color: #000066; font-weight: bold;">true</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">final</span> <span style="color: #000000; font-weight: bold;">class</span> RegistrationForm <span style="color: #000000; font-weight: bold;">extends</span> StatelessForm<span style="color: #339933;">&lt;</span>NewUser<span style="color: #339933;">&gt;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
		<span style="color: #000000; font-weight: bold;">public</span> RegistrationForm<span style="color: #009900;">&#40;</span><span style="color: #003399;">String</span> id<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
			<span style="color: #000000; font-weight: bold;">super</span><span style="color: #009900;">&#40;</span>id, <span style="color: #000000; font-weight: bold;">new</span> CompoundPropertyModel<span style="color: #339933;">&lt;</span>NewUser<span style="color: #339933;">&gt;</span><span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">new</span> NewUser<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
			TextField<span style="color: #339933;">&lt;</span>String<span style="color: #339933;">&gt;</span> emailInput <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">new</span> TextField<span style="color: #339933;">&lt;</span>String<span style="color: #339933;">&gt;</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"email"</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
			add<span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">new</span> InputValidationBorder<span style="color: #339933;">&lt;</span>NewUser<span style="color: #339933;">&gt;</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"validatedEmailBorder"</span>, <span style="color: #000000; font-weight: bold;">this</span>, emailInput<span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
			PasswordTextField passwordInput <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">new</span> PasswordTextField<span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"password"</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
			add<span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">new</span> InputValidationBorder<span style="color: #339933;">&lt;</span>NewUser<span style="color: #339933;">&gt;</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"validatedPasswordBorder"</span>, <span style="color: #000000; font-weight: bold;">this</span>, passwordInput<span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
			PasswordTextField passwordVerificationInput <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">new</span> PasswordTextField<span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"passwordVerification"</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
			add<span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">new</span> InputValidationBorder<span style="color: #339933;">&lt;</span>NewUser<span style="color: #339933;">&gt;</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"validatedPasswordVerificationBorder"</span>, <span style="color: #000000; font-weight: bold;">this</span>, passwordVerificationInput<span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
			TextField<span style="color: #339933;">&lt;</span>Integer<span style="color: #339933;">&gt;</span> ageInput <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">new</span> TextField<span style="color: #339933;">&lt;</span>Integer<span style="color: #339933;">&gt;</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"age"</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
			add<span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">new</span> InputValidationBorder<span style="color: #339933;">&lt;</span>NewUser<span style="color: #339933;">&gt;</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"validatedAgeBorder"</span>, <span style="color: #000000; font-weight: bold;">this</span>, ageInput<span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
			add<span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">new</span> Jsr303FormValidator<span style="color: #009900;">&#40;</span>usernameInput, passwordInput, passwordVerificationInput, ageInput<span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
		<span style="color: #009900;">&#125;</span>
&nbsp;
		@Override
		<span style="color: #000000; font-weight: bold;">protected</span> <span style="color: #000066; font-weight: bold;">void</span> onSubmit<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
			<span style="color: #666666; font-style: italic;">// The NewUser model object is valid!</span>
			<span style="color: #666666; font-style: italic;">// Perform your logic in here...</span>
		<span style="color: #009900;">&#125;</span>
&nbsp;
	<span style="color: #009900;">&#125;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

Now, there is a few things to explain in here:

  * The form has a `NewUser` bean associated. Its code will be shown in the next section.
  * The `InputValidationBorder` encapsulates the functionality to validate an input without validating the full bean and show the validation errors next to that input.
  * The `Jsr303FormValidator` is a form validator. It will only be called when its associated input components are valid (the email, passwords and age) and it will perform a bean scoped validation (in this case, it will check that the 2 supplied passwords are the same). In case it fails, the error will be reported in the panel&#8217;s feedback panel.
  * As the feedback panel should only report the errors that aren&#8217;t associated with a single input, its model is set so that only errors related to RegForm are reported. The only source of these messages will be the Jsr303FormValidator.

### The bean to be validated

The form&#8217;s model is a `NewUser` bean. This bean encapsulates all the data that is requested to a new user. For every property in the bean some constraints must be enforced so they will be annotated following the JSR 303 standard. This is the resulting code `NewUser.java`:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;">@PasswordVerification
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> NewUser <span style="color: #000000; font-weight: bold;">implements</span> <span style="color: #003399;">Serializable</span> <span style="color: #009900;">&#123;</span>
&nbsp;
	<span style="color: #666666; font-style: italic;">// The email</span>
	<span style="color: #000000; font-weight: bold;">private</span> <span style="color: #003399;">String</span> email<span style="color: #339933;">;</span>
&nbsp;
	@NotNull
	@Email
	@Size<span style="color: #009900;">&#40;</span>min<span style="color: #339933;">=</span><span style="color: #cc66cc;">4</span>,max<span style="color: #339933;">=</span><span style="color: #cc66cc;">255</span><span style="color: #009900;">&#41;</span>
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">String</span> getEmail<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">email</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> setEmail<span style="color: #009900;">&#40;</span><span style="color: #003399;">String</span> email<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">email</span> <span style="color: #339933;">=</span> email<span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
	<span style="color: #666666; font-style: italic;">// The password (uncyphered at this stage)</span>
	<span style="color: #000000; font-weight: bold;">private</span> <span style="color: #003399;">String</span> password<span style="color: #339933;">;</span>
&nbsp;
	@NotNull
	@Size<span style="color: #009900;">&#40;</span>min<span style="color: #339933;">=</span><span style="color: #cc66cc;">4</span><span style="color: #009900;">&#41;</span>
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">String</span> getPassword<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">password</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> setPassword<span style="color: #009900;">&#40;</span><span style="color: #003399;">String</span> password<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">password</span> <span style="color: #339933;">=</span> password<span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
	<span style="color: #666666; font-style: italic;">// The password verification</span>
	<span style="color: #000000; font-weight: bold;">private</span> <span style="color: #003399;">String</span> passwordVerification<span style="color: #339933;">;</span>
&nbsp;
	@NotNull
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">String</span> getPasswordVerification<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">passwordVerification</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> setPasswordVerification<span style="color: #009900;">&#40;</span><span style="color: #003399;">String</span> passwordVerification<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">passwordVerification</span> <span style="color: #339933;">=</span> passwordVerification<span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
	<span style="color: #666666; font-style: italic;">// The age</span>
	<span style="color: #000000; font-weight: bold;">private</span> <span style="color: #003399;">Integer</span> age<span style="color: #339933;">;</span>
&nbsp;
	@NotNull
	@Max<span style="color: #009900;">&#40;</span><span style="color: #cc66cc;">140</span><span style="color: #009900;">&#41;</span>
	@Min<span style="color: #009900;">&#40;</span><span style="color: #cc66cc;">18</span><span style="color: #009900;">&#41;</span>
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">Integer</span> getAge<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">age</span><span style="color: #339933;">;</span>
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

The `PasswordVerification` enforces that both supplied passwords match. It will be explained later. The `Email` annotation enforces a valid email address. It is a non-standard constraint part of hibernate validator, but you can easily code a replacement in case you are using a different validating engine and it doesn&#8217;t have that annotation.

### The InputValidationBorder

This border component performs two functions:

  * It encapsulates the input scoped validation logic.
  * And it provides a way to show the related errors close to the input.

It is a generic class whose parameter `T` is the class of the form&#8217;s model object. Its code is as follows:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> InputValidationBorder<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;</span> <span style="color: #000000; font-weight: bold;">extends</span> <span style="color: #003399;">Border</span> <span style="color: #009900;">&#123;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">protected</span> FeedbackPanel feedback<span style="color: #339933;">;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">public</span> InputValidationBorder<span style="color: #009900;">&#40;</span><span style="color: #003399;">String</span> id, <span style="color: #000000; font-weight: bold;">final</span> Form<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;</span> form, <span style="color: #000000; font-weight: bold;">final</span> FormComponent<span style="color: #339933;">&lt;?</span> <span style="color: #000000; font-weight: bold;">extends</span> Object<span style="color: #339933;">&gt;</span> inputComponent<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">super</span><span style="color: #009900;">&#40;</span>id<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
		add<span style="color: #009900;">&#40;</span>inputComponent<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
		inputComponent.<span style="color: #006633;">setRequired</span><span style="color: #009900;">&#40;</span><span style="color: #000066; font-weight: bold;">false</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
		inputComponent.<span style="color: #006633;">add</span><span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">new</span> AjaxFormComponentUpdatingBehavior<span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"onblur"</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
			@Override
			<span style="color: #000000; font-weight: bold;">protected</span> <span style="color: #000066; font-weight: bold;">void</span> onUpdate<span style="color: #009900;">&#40;</span>AjaxRequestTarget target<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
				target.<span style="color: #006633;">addComponent</span><span style="color: #009900;">&#40;</span>InputValidationBorder.<span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">feedback</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
			<span style="color: #009900;">&#125;</span>
&nbsp;
			@Override
			<span style="color: #000000; font-weight: bold;">protected</span> <span style="color: #000066; font-weight: bold;">void</span> onError<span style="color: #009900;">&#40;</span>AjaxRequestTarget target, <span style="color: #003399;">RuntimeException</span> e<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
				target.<span style="color: #006633;">addComponent</span><span style="color: #009900;">&#40;</span>InputValidationBorder.<span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">feedback</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
			<span style="color: #009900;">&#125;</span>
&nbsp;
		<span style="color: #009900;">&#125;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
		inputComponent.<span style="color: #006633;">add</span><span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">new</span> Jsr303PropertyValidator<span style="color: #009900;">&#40;</span>form.<span style="color: #006633;">getModelObject</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">getClass</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>, inputComponent.<span style="color: #006633;">getId</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
		add<span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">feedback</span> <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">new</span> FeedbackPanel<span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"inputErrors"</span>, <span style="color: #000000; font-weight: bold;">new</span> ContainerFeedbackMessageFilter<span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">this</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
		<span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">feedback</span>.<span style="color: #006633;">setOutputMarkupId</span><span style="color: #009900;">&#40;</span><span style="color: #000066; font-weight: bold;">true</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

Again, a few things must be explained:

  * The input component is set to not-required. Bean validation will take care of that constraint in case the property is marked as `@NotNull`.
  * The added `AjaxFormComponentUpdatingBehavior` must override both `onUpdate` and `onError`. In both cases, when the methods are called the validation has already taken place. When the validation fails, `onError` is called, and the feedback component must be in the target to show the error messages. And when the validation succeeds, `onUpdate` is called, and the feedback component must again be in the target, so any older messages get cleared.
  * To save code, a convention where the same name for the input component id&#8217;s and their associated property in `NewUser` is used. Thats the reason the `Jsr303PropertyValidator` is instantiated with the inputComponent&#8217;s id.

The associated markup, `InputValidationBorder.html` is very simple. It just provides a placeholder for the feedback panel next to the input component:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="xml" style="font-family:monospace;"><span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;html</span> <span style="color: #000066;">xmlns:wicket</span>=<span style="color: #ff0000;">"http://wicket.apache.org/dtds.data/wicket-xhtml1.3-strict.dtd"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>  
<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;wicket:border<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;wicket:body</span><span style="color: #000000; font-weight: bold;">/&gt;</span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;span</span> <span style="color: #000066;">wicket:id</span>=<span style="color: #ff0000;">"inputErrors"</span><span style="color: #000000; font-weight: bold;">&gt;</span><span style="color: #000000; font-weight: bold;">&lt;/span<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/wicket:border<span style="color: #000000; font-weight: bold;">&gt;</span></span></span>
<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/html<span style="color: #000000; font-weight: bold;">&gt;</span></span></span></pre>
      </td>
    </tr>
  </table>
</div>

### Jsr303PropertyValidator

This is a custom made validator that enforces the JSR 303 constraints on the indicated bean property. It implements `INullAcceptingValidator` (which extends `IValidator`) so also null values will be passed to the validator.

The validator instance is a Spring supplied bean. It is very easy to <a href="http://cwiki.apache.org/WICKET/spring.html#Spring-AnnotationbasedApproach" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://cwiki.apache.org']);">integrate Wicket with Spring</a>. Just take care that if you must inject dependencies in something that is not a component, you will have to manually call the injector (as it is done in this validator). Also, in case you decide not to use Spring, you can easily change the code to obtain the validator from the Validation class.

The code of `Jsr303PropertyValidator.java` is as follows:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> Jsr303PropertyValidator<span style="color: #339933;">&lt;</span>T, Z<span style="color: #339933;">&gt;</span> <span style="color: #000000; font-weight: bold;">implements</span> INullAcceptingValidator<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
	@SpringBean
	<span style="color: #000000; font-weight: bold;">protected</span> Validator validator<span style="color: #339933;">;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">protected</span> <span style="color: #003399;">String</span> propertyName<span style="color: #339933;">;</span>
	<span style="color: #000000; font-weight: bold;">protected</span> Class<span style="color: #339933;">&lt;</span>Z<span style="color: #339933;">&gt;</span> beanType<span style="color: #339933;">;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">public</span> Jsr303PropertyValidator<span style="color: #009900;">&#40;</span>Class<span style="color: #339933;">&lt;</span>Z<span style="color: #339933;">&gt;</span> clazz, <span style="color: #003399;">String</span> propertyName<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">propertyName</span> <span style="color: #339933;">=</span> propertyName<span style="color: #339933;">;</span>
		<span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">beanType</span> <span style="color: #339933;">=</span> clazz<span style="color: #339933;">;</span>
		injectDependencies<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
	<span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000066; font-weight: bold;">void</span> injectDependencies<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		InjectorHolder.<span style="color: #006633;">getInjector</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">inject</span><span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">this</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
	@Override
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> validate<span style="color: #009900;">&#40;</span>IValidatable<span style="color: #339933;">&lt;</span>T<span style="color: #339933;">&gt;</span> validatable<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		Set<span style="color: #339933;">&lt;</span>ConstraintViolation<span style="color: #339933;">&lt;</span>Z<span style="color: #339933;">&gt;&gt;</span> res <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">validator</span>.<span style="color: #006633;">validateValue</span><span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">beanType</span>, <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">propertyName</span>, validatable.<span style="color: #006633;">getValue</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
		<span style="color: #000000; font-weight: bold;">for</span> <span style="color: #009900;">&#40;</span> ConstraintViolation<span style="color: #339933;">&lt;</span>Z<span style="color: #339933;">&gt;</span> vio <span style="color: #339933;">:</span> res <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
			validatable.<span style="color: #006633;">error</span><span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">new</span> ValidationError<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">setMessage</span><span style="color: #009900;">&#40;</span>vio.<span style="color: #006633;">getMessage</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
		<span style="color: #009900;">&#125;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

The class is generic: T is the class of the property to be validated, while Z is the class of the bean which contains the property (in this case, NewUser).

### Jsr303FormValidator

This class implements `IFormValidator`, and it will be called when all the validations for the associated components have succeeded. It performs a full bean validation (not just the class level annotations), so you may use it to enforce individual properties as well. In this example, as all the properties&#8217; constraints get previously validated via the Jsr303PropertyValidator, only the bean scoped constraints can fail.

This is the code of the class:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> Jsr303FormValidator <span style="color: #000000; font-weight: bold;">implements</span> IFormValidator <span style="color: #009900;">&#123;</span>
&nbsp;
	@SpringBean
	<span style="color: #000000; font-weight: bold;">protected</span> Validator validator<span style="color: #339933;">;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000000; font-weight: bold;">final</span> FormComponent<span style="color: #339933;">&lt;?&gt;</span><span style="color: #009900;">&#91;</span><span style="color: #009900;">&#93;</span> components<span style="color: #339933;">;</span>
&nbsp;
&nbsp;
	<span style="color: #000000; font-weight: bold;">public</span> Jsr303FormValidator<span style="color: #009900;">&#40;</span>FormComponent<span style="color: #339933;">&lt;?&gt;</span>...<span style="color: #006633;">components</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">components</span> <span style="color: #339933;">=</span> components<span style="color: #339933;">;</span>
		injectDependencies<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
	<span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000066; font-weight: bold;">void</span> injectDependencies<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		InjectorHolder.<span style="color: #006633;">getInjector</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">inject</span><span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">this</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
	@Override
	<span style="color: #000000; font-weight: bold;">public</span> FormComponent<span style="color: #339933;">&lt;?&gt;</span><span style="color: #009900;">&#91;</span><span style="color: #009900;">&#93;</span> getDependentFormComponents<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">components</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
	@Override
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> validate<span style="color: #009900;">&#40;</span>Form<span style="color: #339933;">&lt;?&gt;</span> form<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
		ConstraintViolation<span style="color: #009900;">&#91;</span><span style="color: #009900;">&#93;</span> res <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">validator</span>.<span style="color: #006633;">validate</span><span style="color: #009900;">&#40;</span>form.<span style="color: #006633;">getModelObject</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">toArray</span><span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">new</span> ConstraintViolation<span style="color: #009900;">&#91;</span><span style="color: #cc66cc;"></span><span style="color: #009900;">&#93;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
		<span style="color: #000000; font-weight: bold;">for</span> <span style="color: #009900;">&#40;</span> ConstraintViolation vio <span style="color: #339933;">:</span> res <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
			form.<span style="color: #006633;">error</span><span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">new</span> ValidationError<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">setMessage</span><span style="color: #009900;">&#40;</span>vio.<span style="color: #006633;">getMessage</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
		<span style="color: #009900;">&#125;</span>
&nbsp;
	<span style="color: #009900;">&#125;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

### The @PasswordVerification constraint

The NewUser bean is annotated with this constraint, that will enforce that the `password` and `passwordVerification` fields are the same. In order to work, it needs both the annotation definition code and the implementation of the validator. This is not really relevant to the integration part, but I provide it so there is a bean scoped constraint and you can check the `Jsr303FormValidator`. Here is the code for the annotation and the validator:

`PasswordVerification.java`

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;">@Documented
@Target<span style="color: #009900;">&#40;</span>ElementType.<span style="color: #006633;">TYPE</span><span style="color: #009900;">&#41;</span>
@Retention<span style="color: #009900;">&#40;</span>RetentionPolicy.<span style="color: #006633;">RUNTIME</span><span style="color: #009900;">&#41;</span>
@Constraint<span style="color: #009900;">&#40;</span>validatedBy <span style="color: #339933;">=</span> PasswordVerificationValidator.<span style="color: #000000; font-weight: bold;">class</span><span style="color: #009900;">&#41;</span>
<span style="color: #000000; font-weight: bold;">public</span> @<span style="color: #000000; font-weight: bold;">interface</span> PasswordVerification <span style="color: #009900;">&#123;</span>
&nbsp;
    <span style="color: #003399;">String</span> message<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #000000; font-weight: bold;">default</span> <span style="color: #0000ff;">"{newuser.passwordverification}"</span><span style="color: #339933;">;</span>
&nbsp;
    Class<span style="color: #339933;">&lt;?&gt;</span><span style="color: #009900;">&#91;</span><span style="color: #009900;">&#93;</span> groups<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #000000; font-weight: bold;">default</span> <span style="color: #009900;">&#123;</span><span style="color: #009900;">&#125;</span><span style="color: #339933;">;</span>
&nbsp;
    Class<span style="color: #339933;">&lt;?</span> <span style="color: #000000; font-weight: bold;">extends</span> Payload<span style="color: #339933;">&gt;</span><span style="color: #009900;">&#91;</span><span style="color: #009900;">&#93;</span> payload<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #000000; font-weight: bold;">default</span> <span style="color: #009900;">&#123;</span><span style="color: #009900;">&#125;</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

`PasswordVerificationValidator.java`

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> PasswordVerificationValidator <span style="color: #000000; font-weight: bold;">implements</span> ConstraintValidator<span style="color: #339933;">&lt;</span>PasswordVerification, NewUser<span style="color: #339933;">&gt;</span><span style="color: #009900;">&#123;</span>
&nbsp;
	@Override
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> initialize<span style="color: #009900;">&#40;</span>PasswordVerification constraintAnnotation<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #666666; font-style: italic;">// Nothing to do</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
	@Override
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">boolean</span> isValid<span style="color: #009900;">&#40;</span>NewUser value, ConstraintValidatorContext context<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">if</span> <span style="color: #009900;">&#40;</span> value.<span style="color: #006633;">getPassword</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #339933;">==</span> <span style="color: #000066; font-weight: bold;">null</span> <span style="color: #339933;">&&</span> value.<span style="color: #006633;">getPasswordVerification</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #339933;">==</span> <span style="color: #000066; font-weight: bold;">null</span> <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
			<span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000066; font-weight: bold;">true</span><span style="color: #339933;">;</span>
		<span style="color: #009900;">&#125;</span>
		<span style="color: #000000; font-weight: bold;">else</span> <span style="color: #000000; font-weight: bold;">if</span> <span style="color: #009900;">&#40;</span> value.<span style="color: #006633;">getPassword</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #339933;">==</span> <span style="color: #000066; font-weight: bold;">null</span> <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
			<span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000066; font-weight: bold;">false</span><span style="color: #339933;">;</span>
		<span style="color: #009900;">&#125;</span>
		<span style="color: #000000; font-weight: bold;">return</span> <span style="color: #009900;">&#40;</span> value.<span style="color: #006633;">getPassword</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">equals</span><span style="color: #009900;">&#40;</span>value.<span style="color: #006633;">getPasswordVerification</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

### Final touches, i18n

With the above code, you have all you need to use JSR 303 Validation in your Wicket forms. You have means to both validate individual properties associated with an input and the whole bean in a form&#8217;s model.

But the example is incomplete if you need your application to be available in various languages. The validation output messages are produced and interpolated by the validation engine, which isn&#8217;t aware of Wicket&#8217;s session locale. To correct this, a new MessageInterpolator which can access Wicket&#8217;s locale will be supplied to the validator bean.

The code of the new message interpolator (`WicketSessionLocaleMessageInterpolator`) is as follows:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">java.util.Locale</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.apache.wicket.Session</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.hibernate.validator.engine.ResourceBundleMessageInterpolator</span><span style="color: #339933;">;</span>
<span style="color: #000000; font-weight: bold;">import</span> <span style="color: #006699;">org.springframework.stereotype.Component</span><span style="color: #339933;">;</span>
&nbsp;
@<span style="color: #003399;">Component</span><span style="color: #009900;">&#40;</span>value<span style="color: #339933;">=</span><span style="color: #0000ff;">"webLocaleInterpolator"</span><span style="color: #009900;">&#41;</span>
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> WicketSessionLocaleMessageInterpolator <span style="color: #000000; font-weight: bold;">extends</span> ResourceBundleMessageInterpolator <span style="color: #009900;">&#123;</span>
&nbsp;
	@Override
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">String</span> interpolate<span style="color: #009900;">&#40;</span><span style="color: #003399;">String</span> message, <span style="color: #003399;">Context</span> context<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">super</span>.<span style="color: #006633;">interpolate</span><span style="color: #009900;">&#40;</span>message, context, Session.<span style="color: #006633;">get</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">getLocale</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
	@Override
	<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">String</span> interpolate<span style="color: #009900;">&#40;</span><span style="color: #003399;">String</span> message, <span style="color: #003399;">Context</span> context, <span style="color: #003399;">Locale</span> locale<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
		<span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">super</span>.<span style="color: #006633;">interpolate</span><span style="color: #009900;">&#40;</span>message, context, Session.<span style="color: #006633;">get</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">getLocale</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
	<span style="color: #009900;">&#125;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

This class extends `ResourceBundleMessageInterpolator` which is especific to Hibernate&#8217;s Validator implementation, but it&#8217;s very likely that if you use a different provider you can code a class similar to this one.

And the last needed step is to provide this bean to the validator declaration in Spring. This is the relevant part of the applicationContext.xml:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="xml" style="font-family:monospace;"><span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;bean</span> <span style="color: #000066;">id</span>=<span style="color: #ff0000;">"validator"</span> <span style="color: #000066;">class</span>=<span style="color: #ff0000;">"org.springframework.validation.beanvalidation.LocalValidatorFactoryBean"</span><span style="color: #000000; font-weight: bold;">&gt;</span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;property</span> <span style="color: #000066;">name</span>=<span style="color: #ff0000;">"messageInterpolator"</span> <span style="color: #000066;">ref</span>=<span style="color: #ff0000;">"webLocaleInterpolator"</span> <span style="color: #000000; font-weight: bold;">/&gt;</span></span>
<span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;/bean<span style="color: #000000; font-weight: bold;">&gt;</span></span></span></pre>
      </td>
    </tr>
  </table>
</div>

Now you have everything set-up: form and individual input validation with AJAX callbacks, and localized messages. Hope it can be of help <img src="http://carinae.net/wp-includes/images/smilies/icon_smile.gif" alt=":-)" class="wp-smiley" />