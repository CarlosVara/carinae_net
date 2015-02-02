---
title: 'Benchmarking Hibernate Validator and Apache BeanValidation: the two JSR-303 implementations'
author: Carlos Vara
layout: post
permalink: /2010/06/benchmarking-hibernate-validator-and-apache-beanvalidation-the-two-jsr-303-implementations/
categories:
  - Java
tags:
  - apache
  - beanvalidation
  - benchmark
  - hibernate-validator
  - jsr-303
---
Until recently, if you decided to use JSR-303 validations on a project, the choice of what implementation to use was an easy one: Hibernate Validator was the reference implementation and the only one that passed the TCK to ensure its correct behavior. But on the 11th of June the Apache BeanValidation team released a <a href="http://incubator.apache.org/bval/cwiki/2010/06/11/apache-bean-validation-01-incubating-released.html" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://incubator.apache.org']);">version of its validator that passes the TCK</a>, providing a new compliant implementation and giving more choice to the end users.

One of the features that can influence what implementation suits your needs better is how well they perform. So, in this post I will analyse the performance of both validators in various operations (validating, parsing the constraints, etc.) and 3 different scenarios (from simple beans to complex beans using inheritance and groups).

Additionally, the usage of the tool used to benchmark the implementations will be described so you can use it to perform benchmarks more suited to your environment.

### The contendants

In case you need more information about the two implementations, here is a brief description of what they have to offer.

#### Apache BeanValidation

Formerly <a href="http://code.google.com/p/agimatec-validation/" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://code.google.com']);">Agimatec Validation</a>, since March 2010 it has migrated to Apache where it is currently under <a href="http://incubator.apache.org/bval/" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://incubator.apache.org']);">incubation</a>. One of its most useful extra features is the ability to perform method validation using the same kind of JSR-303 annotations.  
The benchmarked version is: 0.1-incubating.

#### Hibernate Validator

The reference implementation of the standard, coming from the JBoss people. Amongst other extra features, its 4.1.0 release will allow modifications to the constraints definitions in runtime via its <a href="http://in.relation.to/15699.lace" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://in.relation.to']);">programmatic constraint configuration API</a>.  
The benchmarked version is: 4.1.0.CR1.

### Benchmarking procedure

There are 2 main operations that a validator has to perform. First, it needs to parse the constraints defined in a bean to be able to validate it. If you are following good practices in the usage of the validator (reusing the factory), this operation will only be done once per bean, so its performance, while important, won&#8217;t be critical.  
The second operation is the validation in itself, and so it will be done every time a validation call is performed. For this reason, the performance of this operation is very important.

#### Generating the test cases

In order to be able to programatically test beans with different properties, a tool that autogenerates a set of beans has been created. You can grab its code from the sandbox area in Apache&#8217;s BeanValidation subversion: <a href="http://svn.apache.org/repos/asf/incubator/bval/sandbox/jsr303-impl-bench/" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://svn.apache.org']);">jsr303-impl-bench</a>.

The usage of this generator is simple, you simply specify the values for various parameters like the total number of beans, the amount of valid values or the lower and upper bound for the number of fields in the beans, and the generator will create the source files for those beans and a holder bean that will have an array with all the beans.

To benchmark the implementations, 2 junit tests are provided that will use the generated beans as input. Everything is integrated inside a maven project, so simply calling `mvn test` will generate the beans, compile them and run the tests using them as input.

Additionally, a simple shell script (`runner.sh`) is provided. This script will read .properties files from the current dir that define the overridden parameters for the generator and benchmark a JSR-303 implementation against those scenarios.

#### Benchmarked scenarios

Three different scenarios have been benchmarked with the objective of providing an idea of the performance of the two validators when dealing with simple beans; beans with inheritance and no groups; and beans with both inheritance and groups.

  * A very simple scenario (Scenario 1), which will benchmark against 300 beans with no inheritance (all beans inherit from Object) and no groups.
  * A more complex scenario (Scenario 2), which will benchmark against 300 beans, 30% of which will inherit from one of 5 base beans. Again, no groups will be used.
  * An even more complex scenario (Scenario 3), which will benchmark against 300 beans, 60% of which will inherit from one of 10 base beans, and 60% of the beans will have a group sequence definition and their constraints will use some of the 10 defined groups.

The common properties, constant across all the scenarios, are:

  * 80% of the values will be valid.
  * All beans will have between 4 and 7 annotated basic fields (Strings, Integers and ints currently).
  * All beans will have between 1 and 3 annotated fields which reference other beans.
  * The bean graph (created using the references to other beans), will have fill ratios of 80%, 40% and 20% for the first, second and third level respectively.

You can learn about the configuration options by checking the file: `generator.default.properties`

And for each scenario, four operations will be benchmarked:

  * **Raw validation**: Validating already parsed beans.
  * **Raw parsing**: Parsing the metadata constraints defined in the beans.
  * **Combined**: Validates beans which have not been previously parsed.
  * **MT**: Launches 4 threads which will validate already parsed beans.

Copies of the properties file for each scenario are available here: <a href="http://carinae.net/wp-content/uploads/2010/06/scenarios.zip" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://carinae.net']);">scenarios.zip</a>.

### Results

Benchmarking the above scenarios in my CoreDuo T2300 @ 1.66Ghz and 2GB of RAM produced the following results. (Each scenario is benchmarked using 20 different bean graphs, and every operation is run 50 times. Times are normalized to represent the execution of a single operation for the 300 beans).

#### Scenario 1

<img src="http://carinae.net/wp-content/uploads/2010/06/scenario1.jpg" alt="Results for 300 beans, no inheritance, no groups" title="Scenario 1" width="435" height="266" class="aligncenter size-full wp-image-263" />  
Apache implementation is faster when validating beans, both in single threaded and multithreaded benchmarks. Parsing speed is similar, although Apache is a little faster.

#### Scenario 2

<img src="http://carinae.net/wp-content/uploads/2010/06/scenario2.jpg" alt="Results for 300 beans, 30% inheritance, no groups" title="Scenario 2" width="435" height="266" class="aligncenter size-full wp-image-263" />  
Adding inheritance increases the time spent on parsing and validating, but it must be taken into account that the base beans are also annotated, and so the amount of required work is also bigger.

Results are similar to the first scenario, with the Apache implementation performing better.

#### Scenario 3

<img src="http://carinae.net/wp-content/uploads/2010/06/scenario3.jpg" alt="Results for 300 beans, 60% inheritance, 60% beans with groups" title="Scenario 3" width="435" height="266" class="aligncenter size-full wp-image-263" />  
Parsing time has increased as more work has to be done in this phase (processing groups and more beans with inheritance). Validating time decreases, but that is to be expected as validation will stop once one group in the sequence is found to have constraint violations.

Again, Apache performs better with results similar to the other two scenarios.

### Conclusion

Several conclusions can be extracted from the results:

  * Apache implementation is around 50% faster when validating under these scenarios, and so, it is likely that using it will improve your application&#8217;s performance compared to using Hibernate&#8217;s Validator.
  * Both implementations performance scales equally from 1 thread to a multi-threaded scenario with 4 threads.
  * Parsing time is roughly equivalent, but this shouldn&#8217;t affect your application performance as much as validating performance.
  * The relation between the performance of the two implementations doesn&#8217;t change while going from a simple scenario to a more complex one.

### How to replicate the results / run your own benchmarks

To replicate the results in this post, you can fetch revision 956368 of the jsr303-impl-bench project, unzip the scenario definitions linked above, and execute `runner.sh Apache` and `runner.sh Hibernate` to run the benchmarks.

Also, you may want to benchmark the implementations using a scenario which replicates the one in your application more closely. To do so, simply create a new properties file overriding all the needed variables and run the tests against it.