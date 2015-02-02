---
title: Reliable execution of persistent transactional asynchronous tasks using only Spring
author: Carlos Vara
layout: post
permalink: /2010/05/execution-persistent-transactional-tasks-with-spring/
categories:
  - Java
tags:
  - async tasks
  - quartz
  - spring
---
I recently came across the need of adding the ability of executing asynchronous tasks in a project. The project in question had these needs:

  * Ability to execute transactional tasks in the background (like emailing a generated report to a user).
  * Ability to schedule a task so it doesn&#8217;t get executed sooner than the scheduled date.
  * Tasks execute transactional DB operations, so if the transaction fails, they should retry X times before failing.
  * Reliable execution: if a task is scheduled, it is guaranteed to be executed, even if the server fails or the thread executing it dies.
  * Also, the app is already using a DB for data persistence, so it will be preferable to also use it to store the tasks queue instead of requiring extra infrastructure to manage.
  * No need for a JavaEE application server, Tomcat or Jetty should suffice.

As additional requirement, the project size didn&#8217;t justify using Quartz or JMS, as they added too much complexity and dependencies to solve a problem that only requires a small fraction of the functionality these solutions provide.

So this just left me with the help of Spring. Spring has support for scheduling and task execution, but the provided executors either rely on plain threads/timers or need Quartz. Plain threads/timers are fine for almost all of the needs, but they don&#8217;t cover reliable execution, so if for example the server is rebooted, your tasks would be lost (JavaEE timers can be made persistent, but the project&#8217;s target environment was Tomcat).

So building on Spring task executing capabilities, this solution will add persistent storage to the tasks to ensure reliability in their execution.

### Initial requirements

For this solution to work, you just need to have Spring 3 with working declarative transactions (`@Transactional`). I&#8217;m using JPA2 for persistence and optimistic locking to ensure data integrity. If you are using different technologies, adapting the solution should be just a manner of changing exception catching and modifying the DAOs.

### Configuring Spring

As I said earlier, this solution builds on Spring&#8217;s task executing capabilities. This means that I will use Spring to manage the thread pool needed to manage the asynchronous execution of methods marked by `@Scheduled`. Then, in those methods I will add the necessary logic to manage the actual task execution.

Assuming you have the task schema added to your configuration file, these two lines are the only configuration required to create a thread pool of 10 threads and configure Spring to use that pool to run the annotated methods.

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="xml" style="font-family:monospace;">    <span style="color: #808080; font-style: italic;">&lt;!-- A task scheduler that will call @Scheduled methods --&gt;</span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;task:scheduler</span> <span style="color: #000066;">id</span>=<span style="color: #ff0000;">"myScheduler"</span> <span style="color: #000066;">pool-size</span>=<span style="color: #ff0000;">"10"</span><span style="color: #000000; font-weight: bold;">/&gt;</span></span>
    <span style="color: #009900;"><span style="color: #000000; font-weight: bold;">&lt;task:annotation-driven</span> <span style="color: #000066;">scheduler</span>=<span style="color: #ff0000;">"myScheduler"</span><span style="color: #000000; font-weight: bold;">/&gt;</span></span></pre>
      </td>
    </tr>
  </table>
</div>

### A holder class to store the tasks in a queue

Tasks need to be persisted, and in their persisted status they need to carry some extra information to be able to correctly execute them. So each enqueued task will store this:

  * Creation time-stamp: the moment when the task was initially queued for execution.
  * Triggering time-stamp: a task cannot be executed sooner than this.
  * Started time-stamp: the exact moment when a thread starts executing this task.
  * Completed time-stamp: when a task is successfully completed, this gets filled. Along with the started time-stamp, this allows the executor to detect stalled or dead tasks.
  * Serialized task: the actual task.

My JPA2 entity is as follows:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #008000; font-style: italic; font-weight: bold;">/**
 * Persistent entity that stores an async task.
 * 
 * @author Carlos Vara
 */</span>
@<span style="color: #003399;">Entity</span>
@Table<span style="color: #009900;">&#40;</span>name<span style="color: #339933;">=</span><span style="color: #0000ff;">"TASK_QUEUE"</span><span style="color: #009900;">&#41;</span>
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> QueuedTaskHolder <span style="color: #009900;">&#123;</span>
&nbsp;
    <span style="color: #666666; font-style: italic;">// Getters -----------------------------------------------------------------</span>
&nbsp;
    @Id
    @MyAppId
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">String</span> getId<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">if</span> <span style="color: #009900;">&#40;</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">id</span> <span style="color: #339933;">==</span> <span style="color: #000066; font-weight: bold;">null</span> <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
            <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">setId</span><span style="color: #009900;">&#40;</span>UUIDHelper.<span style="color: #006633;">newUUID</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #009900;">&#125;</span>
        <span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">id</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    @NotNull
    @Past
    @Temporal<span style="color: #009900;">&#40;</span>TemporalType.<span style="color: #006633;">TIMESTAMP</span><span style="color: #009900;">&#41;</span>
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">Calendar</span> getCreationStamp<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">creationStamp</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    @Temporal<span style="color: #009900;">&#40;</span>TemporalType.<span style="color: #006633;">TIMESTAMP</span><span style="color: #009900;">&#41;</span>
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">Calendar</span> getTriggerStamp<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">return</span> triggerStamp<span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    @Past
    @Temporal<span style="color: #009900;">&#40;</span>TemporalType.<span style="color: #006633;">TIMESTAMP</span><span style="color: #009900;">&#41;</span>
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">Calendar</span> getStartedStamp<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">startedStamp</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    @Past
    @Temporal<span style="color: #009900;">&#40;</span>TemporalType.<span style="color: #006633;">TIMESTAMP</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span>
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">Calendar</span> getCompletedStamp<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">completedStamp</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    @Lob
    @NotNull
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">byte</span><span style="color: #009900;">&#91;</span><span style="color: #009900;">&#93;</span> getSerializedTask<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">serializedTask</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    @Version
    <span style="color: #000000; font-weight: bold;">protected</span> <span style="color: #000066; font-weight: bold;">int</span> getVersion<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">version</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
    <span style="color: #666666; font-style: italic;">// Setters -----------------------------------------------------------------</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">protected</span> <span style="color: #000066; font-weight: bold;">void</span> setId<span style="color: #009900;">&#40;</span><span style="color: #003399;">String</span> id<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">id</span> <span style="color: #339933;">=</span> id<span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> setCreationStamp<span style="color: #009900;">&#40;</span><span style="color: #003399;">Calendar</span> creationStamp<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">creationStamp</span> <span style="color: #339933;">=</span> creationStamp<span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> setTriggerStamp<span style="color: #009900;">&#40;</span><span style="color: #003399;">Calendar</span> triggerStamp<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">triggerStamp</span> <span style="color: #339933;">=</span> triggerStamp<span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> setStartedStamp<span style="color: #009900;">&#40;</span><span style="color: #003399;">Calendar</span> startedStamp<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">startedStamp</span> <span style="color: #339933;">=</span> startedStamp<span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> setCompletedStamp<span style="color: #009900;">&#40;</span><span style="color: #003399;">Calendar</span> completedStamp<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">completedStamp</span> <span style="color: #339933;">=</span> completedStamp<span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> setSerializedTask<span style="color: #009900;">&#40;</span><span style="color: #000066; font-weight: bold;">byte</span><span style="color: #009900;">&#91;</span><span style="color: #009900;">&#93;</span> serializedTask<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">serializedTask</span> <span style="color: #339933;">=</span> serializedTask<span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> setVersion<span style="color: #009900;">&#40;</span><span style="color: #000066; font-weight: bold;">int</span> version<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">version</span> <span style="color: #339933;">=</span> version<span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
    <span style="color: #666666; font-style: italic;">// Fields ------------------------------------------------------------------</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #003399;">String</span> id<span style="color: #339933;">;</span>
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #003399;">Calendar</span> creationStamp<span style="color: #339933;">;</span>
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #003399;">Calendar</span> triggerStamp <span style="color: #339933;">=</span> <span style="color: #000066; font-weight: bold;">null</span><span style="color: #339933;">;</span>
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #003399;">Calendar</span> startedStamp <span style="color: #339933;">=</span> <span style="color: #000066; font-weight: bold;">null</span><span style="color: #339933;">;</span>
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #003399;">Calendar</span> completedStamp <span style="color: #339933;">=</span> <span style="color: #000066; font-weight: bold;">null</span><span style="color: #339933;">;</span>
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000066; font-weight: bold;">byte</span><span style="color: #009900;">&#91;</span><span style="color: #009900;">&#93;</span> serializedTask<span style="color: #339933;">;</span>
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000066; font-weight: bold;">int</span> version<span style="color: #339933;">;</span>
&nbsp;
&nbsp;
    <span style="color: #666666; font-style: italic;">// Lifecycle events --------------------------------------------------------</span>
&nbsp;
    @SuppressWarnings<span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"unused"</span><span style="color: #009900;">&#41;</span>
    @PrePersist
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000066; font-weight: bold;">void</span> onAbstractBaseEntityPrePersist<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">ensureId</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">markCreation</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * Ensures that the entity has a unique UUID.
     */</span>
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000066; font-weight: bold;">void</span> ensureId<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">getId</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * Sets the creation stamp to now.
     */</span>
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000066; font-weight: bold;">void</span> markCreation<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        setCreationStamp<span style="color: #009900;">&#40;</span><span style="color: #003399;">Calendar</span>.<span style="color: #006633;">getInstance</span><span style="color: #009900;">&#40;</span><span style="color: #003399;">TimeZone</span>.<span style="color: #006633;">getTimeZone</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Etc/UTC"</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
    @Override
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">String</span> toString<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #003399;">SimpleDateFormat</span> sdf <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">new</span> <span style="color: #003399;">SimpleDateFormat</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"yyyy.MM.dd HH:mm:ss z"</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">new</span> ToStringCreator<span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">this</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">append</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"id"</span>, getId<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span>
            .<span style="color: #006633;">append</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"creationStamp"</span>, <span style="color: #009900;">&#40;</span>getCreationStamp<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">!=</span><span style="color: #000066; font-weight: bold;">null</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">?</span>sdf.<span style="color: #006633;">format</span><span style="color: #009900;">&#40;</span>getCreationStamp<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">getTime</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">:</span><span style="color: #000066; font-weight: bold;">null</span><span style="color: #009900;">&#41;</span>
            .<span style="color: #006633;">append</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"startedStamp"</span>, <span style="color: #009900;">&#40;</span>getStartedStamp<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">!=</span><span style="color: #000066; font-weight: bold;">null</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">?</span>sdf.<span style="color: #006633;">format</span><span style="color: #009900;">&#40;</span>getStartedStamp<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">getTime</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">:</span><span style="color: #000066; font-weight: bold;">null</span><span style="color: #009900;">&#41;</span>
            .<span style="color: #006633;">append</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"completedStamp"</span>, <span style="color: #009900;">&#40;</span>getCompletedStamp<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">!=</span><span style="color: #000066; font-weight: bold;">null</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">?</span>sdf.<span style="color: #006633;">format</span><span style="color: #009900;">&#40;</span>getCompletedStamp<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">getTime</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">:</span><span style="color: #000066; font-weight: bold;">null</span><span style="color: #009900;">&#41;</span>
            .<span style="color: #006633;">toString</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>    
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

### A DAO to retrieve tasks from the queue

The executor will do 3 things: enqueue new tasks, get tasks from the queue and execute them, and re-queue tasks that are suspected to be stalled (usually because their executing thread has died). So the DAO has to provide operations to cover those scenarios.

The interface that defines this DAO:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #008000; font-style: italic; font-weight: bold;">/**
 * DAO operations for the {@link QueuedTaskHolder} entities.
 * 
 * @author Carlos Vara
 */</span>
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">interface</span> QueuedTaskHolderDao <span style="color: #009900;">&#123;</span>
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * Adds a new task to the current persistence context. The task will be
     * persisted into the database at flush/commit.
     * 
     * @param queuedTask
     *            The task to be saved (enqueued).
     */</span>
    <span style="color: #000066; font-weight: bold;">void</span> persist<span style="color: #009900;">&#40;</span>QueuedTaskHolder queuedTask<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * Finder that retrieves a task by its id.
     * 
     * @param taskId
     *            The id of the requested task.
     * @return The task with that id, or &lt;code&gt;null&lt;/code&gt; if no such task
     *         exists.
     */</span>
    QueuedTaskHolder findById<span style="color: #009900;">&#40;</span><span style="color: #003399;">String</span> taskId<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * @return A task which is candidate for execution. The receiving thread
     *         will need to ensure a lock on it. &lt;code&gt;null&lt;/code&gt; if no
     *         candidate task is available.
     */</span>
    QueuedTaskHolder findNextTaskForExecution<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * @return A task which has been in execution for too long without
     *         finishing. &lt;code&gt;null&lt;/code&gt; if there aren't stalled tasks.
     */</span>
    QueuedTaskHolder findRandomStalledTask<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

And my JPA2 implementation (I&#8217;m using the new typesafe criteria query):

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #008000; font-style: italic; font-weight: bold;">/**
 * JPA2 implementation of {@link QueuedTaskHolderDao}.
 * 
 * @author Carlos Vara
 */</span>
@<span style="color: #003399;">Repository</span>
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> QueuedTaskHolderDaoJPA2 <span style="color: #000000; font-weight: bold;">implements</span> QueuedTaskHolderDao <span style="color: #009900;">&#123;</span>
&nbsp;
&nbsp;
    <span style="color: #666666; font-style: italic;">// QueuedTaskDao methods ---------------------------------------------------</span>
&nbsp;
    @Override
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> persist<span style="color: #009900;">&#40;</span>QueuedTaskHolder queuedTask<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">entityManager</span>.<span style="color: #006633;">persist</span><span style="color: #009900;">&#40;</span>queuedTask<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    @Override
    <span style="color: #000000; font-weight: bold;">public</span> QueuedTaskHolder findById<span style="color: #009900;">&#40;</span><span style="color: #003399;">String</span> taskId<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">entityManager</span>.<span style="color: #006633;">find</span><span style="color: #009900;">&#40;</span>QueuedTaskHolder.<span style="color: #000000; font-weight: bold;">class</span>, taskId<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    @Override
    <span style="color: #000000; font-weight: bold;">public</span> QueuedTaskHolder findNextTaskForExecution<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
        <span style="color: #003399;">Calendar</span> NOW <span style="color: #339933;">=</span> <span style="color: #003399;">Calendar</span>.<span style="color: #006633;">getInstance</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
        <span style="color: #666666; font-style: italic;">// select qt from QueuedTask where</span>
        <span style="color: #666666; font-style: italic;">//      qt.startedStamp == null AND</span>
        <span style="color: #666666; font-style: italic;">//      (qth.triggerStamp == null || qth.triggerStamp &lt; NOW)</span>
        <span style="color: #666666; font-style: italic;">// order by qth.version ASC, qt.creationStamp ASC</span>
        CriteriaBuilder cb <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">entityManager</span>.<span style="color: #006633;">getCriteriaBuilder</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        CriteriaQuery<span style="color: #339933;">&lt;</span>QueuedTaskHolder<span style="color: #339933;">&gt;</span> cq <span style="color: #339933;">=</span> cb.<span style="color: #006633;">createQuery</span><span style="color: #009900;">&#40;</span>QueuedTaskHolder.<span style="color: #000000; font-weight: bold;">class</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        Root<span style="color: #339933;">&lt;</span>QueuedTaskHolder<span style="color: #339933;">&gt;</span> qth <span style="color: #339933;">=</span> cq.<span style="color: #006633;">from</span><span style="color: #009900;">&#40;</span>QueuedTaskHolder.<span style="color: #000000; font-weight: bold;">class</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        cq.<span style="color: #006633;">select</span><span style="color: #009900;">&#40;</span>qth<span style="color: #009900;">&#41;</span>
            .<span style="color: #006633;">where</span><span style="color: #009900;">&#40;</span>cb.<span style="color: #006633;">and</span><span style="color: #009900;">&#40;</span>cb.<span style="color: #006633;">isNull</span><span style="color: #009900;">&#40;</span>qth.<span style="color: #006633;">get</span><span style="color: #009900;">&#40;</span>QueuedTaskHolder_.<span style="color: #006633;">startedStamp</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span>, 
                    cb.<span style="color: #006633;">or</span><span style="color: #009900;">&#40;</span>
                            cb.<span style="color: #006633;">isNull</span><span style="color: #009900;">&#40;</span>qth.<span style="color: #006633;">get</span><span style="color: #009900;">&#40;</span>QueuedTaskHolder_.<span style="color: #006633;">triggerStamp</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span>,
                            cb.<span style="color: #006633;">lessThan</span><span style="color: #009900;">&#40;</span>qth.<span style="color: #006633;">get</span><span style="color: #009900;">&#40;</span>QueuedTaskHolder_.<span style="color: #006633;">triggerStamp</span><span style="color: #009900;">&#41;</span>, NOW<span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span>
            .<span style="color: #006633;">orderBy</span><span style="color: #009900;">&#40;</span>cb.<span style="color: #006633;">asc</span><span style="color: #009900;">&#40;</span>qth.<span style="color: #006633;">get</span><span style="color: #009900;">&#40;</span>QueuedTaskHolder_.<span style="color: #006633;">version</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span>, cb.<span style="color: #006633;">asc</span><span style="color: #009900;">&#40;</span>qth.<span style="color: #006633;">get</span><span style="color: #009900;">&#40;</span>QueuedTaskHolder_.<span style="color: #006633;">creationStamp</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
        List<span style="color: #339933;">&lt;</span>QueuedTaskHolder<span style="color: #339933;">&gt;</span> results <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">entityManager</span>.<span style="color: #006633;">createQuery</span><span style="color: #009900;">&#40;</span>cq<span style="color: #009900;">&#41;</span>.<span style="color: #006633;">setMaxResults</span><span style="color: #009900;">&#40;</span><span style="color: #cc66cc;">1</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">getResultList</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #000000; font-weight: bold;">if</span> <span style="color: #009900;">&#40;</span> results.<span style="color: #006633;">isEmpty</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
            <span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000066; font-weight: bold;">null</span><span style="color: #339933;">;</span>
        <span style="color: #009900;">&#125;</span>
        <span style="color: #000000; font-weight: bold;">else</span> <span style="color: #009900;">&#123;</span>
            <span style="color: #000000; font-weight: bold;">return</span> results.<span style="color: #006633;">get</span><span style="color: #009900;">&#40;</span><span style="color: #cc66cc;"></span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #009900;">&#125;</span>
&nbsp;
    @Override
    <span style="color: #000000; font-weight: bold;">public</span> QueuedTaskHolder findRandomStalledTask<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
        <span style="color: #003399;">Calendar</span> TOO_LONG_AGO <span style="color: #339933;">=</span> <span style="color: #003399;">Calendar</span>.<span style="color: #006633;">getInstance</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        TOO_LONG_AGO.<span style="color: #006633;">add</span><span style="color: #009900;">&#40;</span><span style="color: #003399;">Calendar</span>.<span style="color: #006633;">SECOND</span>, <span style="color: #339933;">-</span><span style="color: #cc66cc;">7200</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
        <span style="color: #666666; font-style: italic;">// select qth from QueuedTask where </span>
        <span style="color: #666666; font-style: italic;">//      qth.startedStamp != null AND</span>
        <span style="color: #666666; font-style: italic;">//      qth.startedStamp &lt; TOO_LONG_AGO</span>
        CriteriaBuilder cb <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">entityManager</span>.<span style="color: #006633;">getCriteriaBuilder</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        CriteriaQuery<span style="color: #339933;">&lt;</span>QueuedTaskHolder<span style="color: #339933;">&gt;</span> cq <span style="color: #339933;">=</span> cb.<span style="color: #006633;">createQuery</span><span style="color: #009900;">&#40;</span>QueuedTaskHolder.<span style="color: #000000; font-weight: bold;">class</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        Root<span style="color: #339933;">&lt;</span>QueuedTaskHolder<span style="color: #339933;">&gt;</span> qth <span style="color: #339933;">=</span> cq.<span style="color: #006633;">from</span><span style="color: #009900;">&#40;</span>QueuedTaskHolder.<span style="color: #000000; font-weight: bold;">class</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        cq.<span style="color: #006633;">select</span><span style="color: #009900;">&#40;</span>qth<span style="color: #009900;">&#41;</span>.<span style="color: #006633;">where</span><span style="color: #009900;">&#40;</span>
                cb.<span style="color: #006633;">and</span><span style="color: #009900;">&#40;</span>
                        cb.<span style="color: #006633;">isNull</span><span style="color: #009900;">&#40;</span>qth.<span style="color: #006633;">get</span><span style="color: #009900;">&#40;</span>QueuedTaskHolder_.<span style="color: #006633;">completedStamp</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span>,
                        cb.<span style="color: #006633;">lessThan</span><span style="color: #009900;">&#40;</span>qth.<span style="color: #006633;">get</span><span style="color: #009900;">&#40;</span>QueuedTaskHolder_.<span style="color: #006633;">startedStamp</span><span style="color: #009900;">&#41;</span>, TOO_LONG_AGO<span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
        List<span style="color: #339933;">&lt;</span>QueuedTaskHolder<span style="color: #339933;">&gt;</span> stalledTasks <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">entityManager</span>.<span style="color: #006633;">createQuery</span><span style="color: #009900;">&#40;</span>cq<span style="color: #009900;">&#41;</span>.<span style="color: #006633;">getResultList</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
        <span style="color: #000000; font-weight: bold;">if</span> <span style="color: #009900;">&#40;</span> stalledTasks.<span style="color: #006633;">isEmpty</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
            <span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000066; font-weight: bold;">null</span><span style="color: #339933;">;</span>
        <span style="color: #009900;">&#125;</span>
        <span style="color: #000000; font-weight: bold;">else</span> <span style="color: #009900;">&#123;</span>
            <span style="color: #003399;">Random</span> rand <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">new</span> <span style="color: #003399;">Random</span><span style="color: #009900;">&#40;</span><span style="color: #003399;">System</span>.<span style="color: #006633;">currentTimeMillis</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
            <span style="color: #000000; font-weight: bold;">return</span> stalledTasks.<span style="color: #006633;">get</span><span style="color: #009900;">&#40;</span>rand.<span style="color: #006633;">nextInt</span><span style="color: #009900;">&#40;</span>stalledTasks.<span style="color: #006633;">size</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
    <span style="color: #666666; font-style: italic;">// Injected dependencies ---------------------------------------------------</span>
&nbsp;
    @PersistenceContext
    <span style="color: #000000; font-weight: bold;">private</span> EntityManager entityManager<span style="color: #339933;">;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

As it can be seen in the implementation, the &#8220;definitions&#8221; for a stalled task and the priorities given to the tasks in the queue can be easily tweaked in case it&#8217;s needed.

Currently, tasks can be retrieved from the queue as soon as their triggering stamp is reached, and they are ordered by the amount of times they have been tried to execute (a trick using the version column) and by how old they are. It&#8217;s easy to add an extra condition for example to never query tasks that have failed too many times.

### The executor

Now, the most important piece in the system. The executor will:

  * Enqueue (persist) tasks received.
  * Retrieve tasks that need to be executed. Ensure that the current thread gets a proper lock on the task so it&#8217;s the only one attempting its execution.
  * Check for stalled tasks and re-queue them.

The first operation is synchronous, and in my scenario gets executed in the same transaction as the operation that requests the task execution, this way, if for whatever reason the current transaction fails, no spurious tasks are queued.

The other two operations are asynchronous and their execution is managed by the thread pool that was configured in the first step. The rate of execution can be adjusted depending on the amount of tasks that your system needs to manage. Also, these methods will execute/re-queue as many tasks as they can while they have work to do, so there is no need for setting the rates too high.

The executor implements Spring&#8217;s `TaskExecutor` interface, so it can be easily substituted by another implementation should the need for it arise.

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #008000; font-style: italic; font-weight: bold;">/**
 * A task executor with persistent task queueing.
 * 
 * @author Carlos Vara
 */</span>
@<span style="color: #003399;">Component</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"MyTaskExecutor"</span><span style="color: #009900;">&#41;</span>
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> MyTaskExecutor <span style="color: #000000; font-weight: bold;">implements</span> TaskExecutor <span style="color: #009900;">&#123;</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">final</span> <span style="color: #000000; font-weight: bold;">static</span> Logger logger <span style="color: #339933;">=</span> LoggerFactory.<span style="color: #006633;">getLogger</span><span style="color: #009900;">&#40;</span>MyTaskExecutor.<span style="color: #000000; font-weight: bold;">class</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
    @Autowired
    <span style="color: #000000; font-weight: bold;">protected</span> QueuedTaskHolderDao queuedTaskDao<span style="color: #339933;">;</span>
&nbsp;
    @Autowired
    <span style="color: #000000; font-weight: bold;">protected</span> Serializer serializer<span style="color: #339933;">;</span>
&nbsp;
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * Additional requirement: must be run inside a transaction.
     * Currently using MANDATORY as the app won't create tasks outside a
     * transaction.
     * 
     * @see org.springframework.core.task.TaskExecutor#execute(java.lang.Runnable)
     */</span>
    @Override
    @Transactional<span style="color: #009900;">&#40;</span>propagation<span style="color: #339933;">=</span>Propagation.<span style="color: #006633;">MANDATORY</span><span style="color: #009900;">&#41;</span>
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> execute<span style="color: #009900;">&#40;</span><span style="color: #003399;">Runnable</span> task<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
        logger.<span style="color: #006633;">debug</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Trying to enqueue: {}"</span>, task<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
        AbstractBaseTask abt<span style="color: #339933;">;</span> 
        <span style="color: #000000; font-weight: bold;">try</span> <span style="color: #009900;">&#123;</span>
            abt <span style="color: #339933;">=</span> AbstractBaseTask.<span style="color: #000000; font-weight: bold;">class</span>.<span style="color: #006633;">cast</span><span style="color: #009900;">&#40;</span>task<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #009900;">&#125;</span> <span style="color: #000000; font-weight: bold;">catch</span> <span style="color: #009900;">&#40;</span><span style="color: #003399;">ClassCastException</span> e<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
            logger.<span style="color: #006633;">error</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Only runnables that extend AbstractBaseTask are accepted."</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
            <span style="color: #000000; font-weight: bold;">throw</span> <span style="color: #000000; font-weight: bold;">new</span> <span style="color: #003399;">IllegalArgumentException</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Invalid task: "</span> <span style="color: #339933;">+</span> task<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #009900;">&#125;</span>
&nbsp;
        <span style="color: #666666; font-style: italic;">// Serialize the task</span>
        QueuedTaskHolder newTask <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">new</span> QueuedTaskHolder<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #000066; font-weight: bold;">byte</span><span style="color: #009900;">&#91;</span><span style="color: #009900;">&#93;</span> serializedTask <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">serializer</span>.<span style="color: #006633;">serializeObject</span><span style="color: #009900;">&#40;</span>abt<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        newTask.<span style="color: #006633;">setTriggerStamp</span><span style="color: #009900;">&#40;</span>abt.<span style="color: #006633;">getTriggerStamp</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
        logger.<span style="color: #006633;">debug</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"New serialized task takes {} bytes"</span>, serializedTask.<span style="color: #006633;">length</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
        newTask.<span style="color: #006633;">setSerializedTask</span><span style="color: #009900;">&#40;</span>serializedTask<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
        <span style="color: #666666; font-style: italic;">// Store it in the db</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">queuedTaskDao</span>.<span style="color: #006633;">persist</span><span style="color: #009900;">&#40;</span>newTask<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
        <span style="color: #666666; font-style: italic;">// POST: Task has been enqueued</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * Runs enqueued tasks.
     */</span>
    @Scheduled<span style="color: #009900;">&#40;</span>fixedRate<span style="color: #339933;">=</span>60l<span style="color: #339933;">*</span>1000l<span style="color: #009900;">&#41;</span> <span style="color: #666666; font-style: italic;">// Every minute</span>
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> runner<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
        logger.<span style="color: #006633;">debug</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Started runner {}"</span>, <span style="color: #003399;">Thread</span>.<span style="color: #006633;">currentThread</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">getName</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
        QueuedTaskHolder lockedTask <span style="color: #339933;">=</span> <span style="color: #000066; font-weight: bold;">null</span><span style="color: #339933;">;</span>
&nbsp;
        <span style="color: #666666; font-style: italic;">// While there is work to do...</span>
        <span style="color: #000000; font-weight: bold;">while</span> <span style="color: #009900;">&#40;</span> <span style="color: #009900;">&#40;</span>lockedTask <span style="color: #339933;">=</span> tryLockTask<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span> <span style="color: #339933;">!=</span> <span style="color: #000066; font-weight: bold;">null</span> <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
            logger.<span style="color: #006633;">debug</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Obtained lock on {}"</span>, lockedTask<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
            <span style="color: #666666; font-style: italic;">// Deserialize the task</span>
            AbstractBaseTask runnableTask <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">serializer</span>.<span style="color: #006633;">deserializeAndCast</span><span style="color: #009900;">&#40;</span>lockedTask.<span style="color: #006633;">getSerializedTask</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
            runnableTask.<span style="color: #006633;">setQueuedTaskId</span><span style="color: #009900;">&#40;</span>lockedTask.<span style="color: #006633;">getId</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
            <span style="color: #666666; font-style: italic;">// Run it</span>
            runnableTask.<span style="color: #006633;">run</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #009900;">&#125;</span>
&nbsp;
        logger.<span style="color: #006633;">debug</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Finishing runner {}, nothing else to do."</span>, <span style="color: #003399;">Thread</span>.<span style="color: #006633;">currentThread</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">getName</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * The hypervisor re-queues for execution possible stalled tasks.
     */</span>
    @Scheduled<span style="color: #009900;">&#40;</span>fixedRate<span style="color: #339933;">=</span>60l<span style="color: #339933;">*</span>60l<span style="color: #339933;">*</span>1000l<span style="color: #009900;">&#41;</span> <span style="color: #666666; font-style: italic;">// Every hour</span>
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> hypervisor<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
        logger.<span style="color: #006633;">debug</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Started hypervisor {}"</span>, <span style="color: #003399;">Thread</span>.<span style="color: #006633;">currentThread</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">getName</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
        <span style="color: #666666; font-style: italic;">// Reset stalled threads, one at a time to avoid too wide transactions</span>
        <span style="color: #000000; font-weight: bold;">while</span> <span style="color: #009900;">&#40;</span> tryResetStalledTask<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
        logger.<span style="color: #006633;">debug</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Finishing hypervisor {}, nothing else to do."</span>, <span style="color: #003399;">Thread</span>.<span style="color: #006633;">currentThread</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span>.<span style="color: #006633;">getName</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * Tries to ensure a lock on a task in order to execute it.
     * 
     * @return A locked task, or &lt;code&gt;null&lt;/code&gt; if there is no task available
     *         or no lock could be obtained.
     */</span>
    <span style="color: #000000; font-weight: bold;">private</span> QueuedTaskHolder tryLockTask<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
        <span style="color: #000066; font-weight: bold;">int</span> tries <span style="color: #339933;">=</span> <span style="color: #cc66cc;">3</span><span style="color: #339933;">;</span>
&nbsp;
        QueuedTaskHolder ret <span style="color: #339933;">=</span> <span style="color: #000066; font-weight: bold;">null</span><span style="color: #339933;">;</span>
        <span style="color: #000000; font-weight: bold;">while</span> <span style="color: #009900;">&#40;</span> tries <span style="color: #339933;">&gt;</span> <span style="color: #cc66cc;"></span> <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
            <span style="color: #000000; font-weight: bold;">try</span> <span style="color: #009900;">&#123;</span>
                ret <span style="color: #339933;">=</span> obtainLockedTask<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
                <span style="color: #000000; font-weight: bold;">return</span> ret<span style="color: #339933;">;</span>
            <span style="color: #009900;">&#125;</span> <span style="color: #000000; font-weight: bold;">catch</span> <span style="color: #009900;">&#40;</span>OptimisticLockingFailureException e<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
                tries<span style="color: #339933;">--;</span>
            <span style="color: #009900;">&#125;</span>
        <span style="color: #009900;">&#125;</span>
&nbsp;
        <span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000066; font-weight: bold;">null</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * Tries to reset a stalled task.
     * 
     * @return &lt;code&gt;true&lt;/code&gt; if one task was successfully re-queued,
     *         &lt;code&gt;false&lt;/code&gt; if no task was re-queued, either because there
     *         are no stalled tasks or because there was a conflict re-queueing
     *         it.
     */</span>
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000066; font-weight: bold;">boolean</span> tryResetStalledTask<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000066; font-weight: bold;">int</span> tries <span style="color: #339933;">=</span> <span style="color: #cc66cc;">3</span><span style="color: #339933;">;</span>
&nbsp;
        QueuedTaskHolder qt <span style="color: #339933;">=</span> <span style="color: #000066; font-weight: bold;">null</span><span style="color: #339933;">;</span>
        <span style="color: #000000; font-weight: bold;">while</span> <span style="color: #009900;">&#40;</span> tries <span style="color: #339933;">&gt;</span> <span style="color: #cc66cc;"></span> <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
            <span style="color: #000000; font-weight: bold;">try</span> <span style="color: #009900;">&#123;</span>
                qt <span style="color: #339933;">=</span> resetStalledTask<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
                <span style="color: #000000; font-weight: bold;">return</span> qt <span style="color: #339933;">!=</span> <span style="color: #000066; font-weight: bold;">null</span><span style="color: #339933;">;</span>
            <span style="color: #009900;">&#125;</span> <span style="color: #000000; font-weight: bold;">catch</span> <span style="color: #009900;">&#40;</span>OptimisticLockingFailureException e<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
                tries<span style="color: #339933;">--;</span>
            <span style="color: #009900;">&#125;</span>
        <span style="color: #009900;">&#125;</span>
&nbsp;
        <span style="color: #000000; font-weight: bold;">return</span> <span style="color: #000066; font-weight: bold;">false</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * @return A locked task ready for execution, &lt;code&gt;null&lt;/code&gt; if no ready
     *         task is available.
     * @throws OptimisticLockingFailureException
     *             If getting the lock fails.
     */</span>
    @Transactional
    <span style="color: #000000; font-weight: bold;">public</span> QueuedTaskHolder obtainLockedTask<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        QueuedTaskHolder qt <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">queuedTaskDao</span>.<span style="color: #006633;">findNextTaskForExecution</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        logger.<span style="color: #006633;">debug</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Next possible task for execution {}"</span>, qt<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #000000; font-weight: bold;">if</span> <span style="color: #009900;">&#40;</span> qt <span style="color: #339933;">!=</span> <span style="color: #000066; font-weight: bold;">null</span> <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
            qt.<span style="color: #006633;">setStartedStamp</span><span style="color: #009900;">&#40;</span><span style="color: #003399;">Calendar</span>.<span style="color: #006633;">getInstance</span><span style="color: #009900;">&#40;</span><span style="color: #003399;">TimeZone</span>.<span style="color: #006633;">getTimeZone</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"etc/UTC"</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #009900;">&#125;</span>
        <span style="color: #000000; font-weight: bold;">return</span> qt<span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * Tries to reset a stalled task, returns null if no stalled task was reset.
     * 
     * @return The re-queued task, &lt;code&gt;null&lt;/code&gt; if no stalled task is
     *         available.
     * @throws OptimisticLockingFailureException
     *             If the stalled task is modified by another thread during
     *             re-queueing.
     */</span>
    @Transactional
    <span style="color: #000000; font-weight: bold;">public</span> QueuedTaskHolder resetStalledTask<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        QueuedTaskHolder stalledTask <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">queuedTaskDao</span>.<span style="color: #006633;">findRandomStalledTask</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        logger.<span style="color: #006633;">debug</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Obtained this stalledTask {}"</span>, stalledTask<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #000000; font-weight: bold;">if</span> <span style="color: #009900;">&#40;</span> stalledTask <span style="color: #339933;">!=</span> <span style="color: #000066; font-weight: bold;">null</span> <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
            stalledTask.<span style="color: #006633;">setStartedStamp</span><span style="color: #009900;">&#40;</span><span style="color: #000066; font-weight: bold;">null</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #009900;">&#125;</span>
        <span style="color: #000000; font-weight: bold;">return</span> stalledTask<span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

### The base task and an example task

Now, to ensure the correct transactionality of the task execution, and that they get correctly de-queued upon completion, some extra work has to be done during their execution. This extra functionality will be centralized in a base abstract task class, from whom all the tasks in the system will inherit.

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #008000; font-style: italic; font-weight: bold;">/**
 * Superclass for all async tasks.
 * &lt;ul&gt;
 *  &lt;li&gt;Ensures that its associated queued task is marked as completed in the same tx.&lt;/li&gt;
 *  &lt;li&gt;Marks the task as serializable.&lt;/li&gt;
 * &lt;/ul&gt;
 * 
 * @author Carlos Vara
 */</span>
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">abstract</span> <span style="color: #000000; font-weight: bold;">class</span> AbstractBaseTask <span style="color: #000000; font-weight: bold;">implements</span> <span style="color: #003399;">Runnable</span>, <span style="color: #003399;">Serializable</span> <span style="color: #009900;">&#123;</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">final</span> <span style="color: #000000; font-weight: bold;">static</span> Logger logger <span style="color: #339933;">=</span> LoggerFactory.<span style="color: #006633;">getLogger</span><span style="color: #009900;">&#40;</span>AbstractBaseTask.<span style="color: #000000; font-weight: bold;">class</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
&nbsp;
    <span style="color: #666666; font-style: italic;">// Common data -------------------------------------------------------------</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000000; font-weight: bold;">transient</span> <span style="color: #003399;">String</span> queuedTaskId<span style="color: #339933;">;</span>
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000000; font-weight: bold;">transient</span> QueuedTaskHolder qth<span style="color: #339933;">;</span>
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000000; font-weight: bold;">transient</span> <span style="color: #003399;">Calendar</span> triggerStamp<span style="color: #339933;">;</span>
&nbsp;
&nbsp;
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> setQueuedTaskId<span style="color: #009900;">&#40;</span><span style="color: #003399;">String</span> queuedTaskId<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">queuedTaskId</span> <span style="color: #339933;">=</span> queuedTaskId<span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">String</span> getQueuedTaskId<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">return</span> queuedTaskId<span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> setTriggerStamp<span style="color: #009900;">&#40;</span><span style="color: #003399;">Calendar</span> triggerStamp<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">triggerStamp</span> <span style="color: #339933;">=</span> triggerStamp<span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #003399;">Calendar</span> getTriggerStamp<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">return</span> triggerStamp<span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
    <span style="color: #666666; font-style: italic;">// Injected components -----------------------------------------------------</span>
&nbsp;
    @Autowired<span style="color: #009900;">&#40;</span>required<span style="color: #339933;">=</span><span style="color: #000066; font-weight: bold;">true</span><span style="color: #009900;">&#41;</span>
    <span style="color: #000000; font-weight: bold;">protected</span> <span style="color: #000000; font-weight: bold;">transient</span> QueuedTaskHolderDao queuedTaskHolderDao<span style="color: #339933;">;</span>
&nbsp;
&nbsp;
    <span style="color: #666666; font-style: italic;">// Lifecycle methods -------------------------------------------------------</span>
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * Entrance point of the task.
     * &lt;ul&gt;
     *  &lt;li&gt;Ensures that the associated task in the queue exists.&lt;/li&gt;
     *  &lt;li&gt;Marks the queued task as finished upon tx commit.&lt;/li&gt;
     *  &lt;li&gt;In case of tx rollback, frees the task.&lt;/li&gt;
     * &lt;/ul&gt;
     * 
     * @see java.lang.Runnable#run()
     */</span>
    @Override
    <span style="color: #000000; font-weight: bold;">final</span> <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> run<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
        <span style="color: #000000; font-weight: bold;">try</span> <span style="color: #009900;">&#123;</span>
            transactionalOps<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #009900;">&#125;</span> <span style="color: #000000; font-weight: bold;">catch</span> <span style="color: #009900;">&#40;</span><span style="color: #003399;">RuntimeException</span> e<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
            <span style="color: #666666; font-style: italic;">// Free the task, so it doesn't stall</span>
            logger.<span style="color: #006633;">warn</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Exception forced task tx rollback: {}"</span>, e<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
            freeTask<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #009900;">&#125;</span>
&nbsp;
    @Transactional
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000066; font-weight: bold;">void</span> transactionalOps<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        doInTxBeforeTask<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        doTaskInTransaction<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        doInTxAfterTask<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    @Transactional
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000066; font-weight: bold;">void</span> freeTask<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        QueuedTaskHolder task <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">queuedTaskHolderDao</span>.<span style="color: #006633;">findById</span><span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">queuedTaskId</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        task.<span style="color: #006633;">setStartedStamp</span><span style="color: #009900;">&#40;</span><span style="color: #000066; font-weight: bold;">null</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * Ensures that there is an associated task and that its state is valid.
     * Shouldn't be needed, just for extra security.
     */</span>
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000066; font-weight: bold;">void</span> doInTxBeforeTask<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">qth</span> <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">queuedTaskHolderDao</span>.<span style="color: #006633;">findById</span><span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">queuedTaskId</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #000000; font-weight: bold;">if</span> <span style="color: #009900;">&#40;</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">qth</span> <span style="color: #339933;">==</span> <span style="color: #000066; font-weight: bold;">null</span> <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
            <span style="color: #000000; font-weight: bold;">throw</span> <span style="color: #000000; font-weight: bold;">new</span> <span style="color: #003399;">IllegalArgumentException</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Not executing: no associated task exists: "</span> <span style="color: #339933;">+</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">getQueuedTaskId</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #009900;">&#125;</span>
        <span style="color: #000000; font-weight: bold;">if</span> <span style="color: #009900;">&#40;</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">qth</span>.<span style="color: #006633;">getStartedStamp</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #339933;">==</span> <span style="color: #000066; font-weight: bold;">null</span> <span style="color: #339933;">||</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">qth</span>.<span style="color: #006633;">getCompletedStamp</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #339933;">!=</span> <span style="color: #000066; font-weight: bold;">null</span> <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
            <span style="color: #000000; font-weight: bold;">throw</span> <span style="color: #000000; font-weight: bold;">new</span> <span style="color: #003399;">IllegalArgumentException</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"Illegal queued task status: "</span> <span style="color: #339933;">+</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">qth</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #009900;">&#125;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * Method to be implemented by concrete tasks where their operations are
     * performed.
     */</span>
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">abstract</span> <span style="color: #000066; font-weight: bold;">void</span> doTaskInTransaction<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * Marks the associated task as finished.
     */</span>
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000066; font-weight: bold;">void</span> doInTxAfterTask<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">qth</span>.<span style="color: #006633;">setCompletedStamp</span><span style="color: #009900;">&#40;</span><span style="color: #003399;">Calendar</span>.<span style="color: #006633;">getInstance</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000000; font-weight: bold;">static</span> <span style="color: #000000; font-weight: bold;">final</span> <span style="color: #000066; font-weight: bold;">long</span> serialVersionUID <span style="color: #339933;">=</span> 1L<span style="color: #339933;">;</span>
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

The class also holds a trigger stamp field that can be used before calling `MyTaskExecutor.execute()` to schedule the task for a given time and date.

A simple (and useless) example task that extends this base task:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;"><span style="color: #008000; font-style: italic; font-weight: bold;">/**
 * Logs the status of a User.
 * 
 * @author Carlos Vara
 */</span>
@Configurable
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> ExampleTask <span style="color: #000000; font-weight: bold;">extends</span> AbstractBountyTask <span style="color: #009900;">&#123;</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">final</span> <span style="color: #000000; font-weight: bold;">static</span> Logger logger <span style="color: #339933;">=</span> LoggerFactory.<span style="color: #006633;">getLogger</span><span style="color: #009900;">&#40;</span>ExampleTask.<span style="color: #000000; font-weight: bold;">class</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
&nbsp;
&nbsp;
    <span style="color: #666666; font-style: italic;">// Injected components -----------------------------------------------------</span>
&nbsp;
    @Autowired
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000000; font-weight: bold;">transient</span> UserDao userDao<span style="color: #339933;">;</span>
&nbsp;
&nbsp;
    <span style="color: #666666; font-style: italic;">// Data --------------------------------------------------------------------</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000000; font-weight: bold;">final</span> <span style="color: #003399;">String</span> userId<span style="color: #339933;">;</span>
&nbsp;
&nbsp;
    <span style="color: #000000; font-weight: bold;">public</span> ExampleTask<span style="color: #009900;">&#40;</span><span style="color: #003399;">String</span> userId<span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">userId</span> <span style="color: #339933;">=</span> userId<span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
&nbsp;
    <span style="color: #008000; font-style: italic; font-weight: bold;">/**
     * Logs the status of a user.
     */</span>
    @Override
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> doTaskInTransaction<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
&nbsp;
        <span style="color: #666666; font-style: italic;">// Get the user</span>
        User user <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">userDao</span>.<span style="color: #006633;">findBagById</span><span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">userId</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #000000; font-weight: bold;">if</span> <span style="color: #009900;">&#40;</span> user <span style="color: #339933;">==</span> <span style="color: #000066; font-weight: bold;">null</span> <span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
            logger.<span style="color: #006633;">error</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"User {} doesn't exist in the system."</span>, userId<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
            <span style="color: #000000; font-weight: bold;">return</span><span style="color: #339933;">;</span>
        <span style="color: #009900;">&#125;</span>
&nbsp;
        <span style="color: #666666; font-style: italic;">// Log the user status</span>
        logger.<span style="color: #006633;">info</span><span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"User status: {}"</span>, user.<span style="color: #006633;">getStatus</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
&nbsp;
    <span style="color: #000000; font-weight: bold;">private</span> <span style="color: #000000; font-weight: bold;">static</span> <span style="color: #000000; font-weight: bold;">final</span> <span style="color: #000066; font-weight: bold;">long</span> serialVersionUID <span style="color: #339933;">=</span> 1L<span style="color: #339933;">;</span>
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

It&#8217;s important to note that I&#8217;m using Spring&#8217;s `@Configurable` to manage dependency injection after the tasks have been deserialized. You can solve this problem in a different way if using aspectj isn&#8217;t a possibility.

### And finally, an example of how to use it

Last thing, a piece of simple code that shows how to send a task to the background to be executed as soon as possible and how to schedule a task so it will be executed the next day:

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="java" style="font-family:monospace;">@Service
<span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000000; font-weight: bold;">class</span> ExampleServiceImpl <span style="color: #000000; font-weight: bold;">implements</span> ExampleService <span style="color: #009900;">&#123;</span>
&nbsp;
    @Qualifier<span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"BountyExecutor"</span><span style="color: #009900;">&#41;</span>
    @Autowired
    <span style="color: #000000; font-weight: bold;">private</span> TaskExecutor taskExecutor<span style="color: #339933;">;</span>
&nbsp;
    @Transactional
    <span style="color: #000000; font-weight: bold;">public</span> <span style="color: #000066; font-weight: bold;">void</span> example<span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span> <span style="color: #009900;">&#123;</span>
        <span style="color: #666666; font-style: italic;">// Task will execute ASAP</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">taskExecutor</span>.<span style="color: #006633;">execute</span><span style="color: #009900;">&#40;</span><span style="color: #000000; font-weight: bold;">new</span> ExampleTask<span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"1"</span><span style="color: #009900;">&#41;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #666666; font-style: italic;">// Task won't execute until tomorrow</span>
        ExampleTask et <span style="color: #339933;">=</span> <span style="color: #000000; font-weight: bold;">new</span> ExampleTask<span style="color: #009900;">&#40;</span><span style="color: #0000ff;">"2"</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #003399;">Calendar</span> tomorrow <span style="color: #339933;">=</span> <span style="color: #003399;">Calendar</span>.<span style="color: #006633;">getInstance</span><span style="color: #009900;">&#40;</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        tomorrow.<span style="color: #006633;">add</span><span style="color: #009900;">&#40;</span><span style="color: #003399;">Calendar</span>.<span style="color: #006633;">DAY</span>, <span style="color: #cc66cc;">1</span><span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        et.<span style="color: #006633;">setTriggerStamp</span><span style="color: #009900;">&#40;</span>tomorrow<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
        <span style="color: #000000; font-weight: bold;">this</span>.<span style="color: #006633;">taskExecutor</span>.<span style="color: #006633;">execute</span><span style="color: #009900;">&#40;</span>et<span style="color: #009900;">&#41;</span><span style="color: #339933;">;</span>
    <span style="color: #009900;">&#125;</span>
<span style="color: #009900;">&#125;</span></pre>
      </td>
    </tr>
  </table>
</div>

### An explanation of a task lifetime

Given that the algorithm presented here is a bit complex, I will details the steps in the lifetime of a task to clarify how the system ensures reliable execution.

#### Step 1: task queuing

A task is enqueued when calling MyTaskExecutor.execute(). The en-queuing is part of the transaction opened in the service method that creates the task, so if that transaction fails, both your service method changes and the task data are left uncommitted, which is the correct behavior.

#### Step 2: task locking

Your task is stored in the DB, and it has its started and completed stamps set to null. This means that it hasn&#8217;t been executed yet, and that it seems that nobody is trying to execute it. The executor then tries to lock it, by fetching it from the db and setting its started stamp. If that transaction succeeds, it&#8217;s guaranteed that the thread is the only one with that task assigned. If the thread were to die now, in between transactions, the task would eventually become stalled and be re-queued by the hypervisor.

#### Step 3: task execution

Now that the thread has a lock in the task, the execution starts. A new transaction is started, and the task operations are performed inside it along with marking the task as completed at the end of the transaction. If the transaction succeeds, the task will be correctly de-queued as part of it. If it fails, a try is done to free the task immediately, but if this try also failed (or its code was never reached) the task would be eventually collected by the hypervisor.

And that&#8217;s it. Hope you find it useful, please post a comment if you successfully re-use the system <img src="http://carinae.net/wp-includes/images/smilies/icon_smile.gif" alt=":-)" class="wp-smiley" />

**Edit: 2010-07-05**  
I shared a template project which illustrates this system at github: <a href="http://github.com/CarlosVara/spring-async-persistent-tasks" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://github.com']);">http://github.com/CarlosVara/spring-async-persistent-tasks</a>
