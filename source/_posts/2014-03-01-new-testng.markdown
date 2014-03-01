---
layout: post
title: "浅谈Testng测试框架的分布式实现"
date: 2014-03-01 22:23:57 +0800
comments: true
categories: 
---

### 现状分析

[TestNG](http://testng.org)是目前在Java领域用的比较多的一款测试框架，它是基于JUnit的一个二次封装，不过增加了很多新的功能，比如完全基于Annotaion的配置，测试用例的依赖等等。你可以通过review它的Guide文档来获得更多的信息，不够有一个问题就是，TestNG的文档里面之字未提关于分布式执行测试用例方面的内容，难道TestNG不支持分布式测试么？错，其实作者老早就写了一篇Blog来介绍如何使用TestNG的分布式了，详情请见[TestNG Distribute](http://beust.com/weblog2/archives/000362.html)。但是如果你拿最新的Testng6.8.7的版本来配置分布式的实现，肯定会遇到一堆的问题，所以结论就是目前的分布式是有Bug存在的。

### 问题在哪里

既然有问题，我们就开始尝试从源码的角度来查找问题了。TestNG的源码目前托管在[GitHub](https://github.com/cbeust/testng)上,下载下来以后，只要把其中的ivy.jar复制到ant的lib目录下，然后修改一下build.properties的"yaml.jar=snakeyaml-1.6.jar"到"yaml.jar=snakeyaml-1.12.jar"，然后直接ant build就可以编译一个定制化的TestNG了。TestNG和分布式相关的一系列相关code在remote的包里面。经过阅读，我们发现了一些问题。

1. org.testng.remote.RemoteSuiteWorker

``` java
public class RemoteSuiteWorker extends RemoteWorker implements Runnable {
  private XmlSuite m_suite;

  public RemoteSuiteWorker(XmlSuite suite, SlavePool slavePool, RemoteResultListener listener) {
    super(listener, slavePool);
    m_suite = suite;
  }

  @Override
  public void run() {
    try {
      SuiteRunner result = sendSuite(getSlavePool().getSlave(), m_suite);
      m_listener.onResult(result);
    }
    catch (Exception e) {
      e.printStackTrace();
    }

  }
}

```
每一个RemoteSuiteWorker是一个线程，每一个Thread对应一个Slave机器，然后通过Socket通信把一个Xmluite对象发送到Slave端进行执行。TestNG通过一个SlavePool来管理所有的链接，但是我们在这里就能看到一个问题，这个连接池的实现缺少了一个把Socket连接返回到SlavePool的操作，所以说一旦case执行的数量超过了Slave的机器的数量，那么后面一旦SlavePool枯竭了，那么整个分布式执行就会Block在了SlavePool这里了。对了，SlavePool是一个阻塞的双端队列的实现。

2. org.testng.remote.adapter.DefaultMasterAdapter

``` java

public void awaitTermination(long timeout) throws InterruptedException
	{
		ThreadUtil.execute(m_workers, 1, 10 * 1000L, false);
	}

```

当Master进程把所有的测试用例分发到一个队列以后，就需要把这个队列放到一个多线程里面去执行，上面这个方法就是需要去执行所有线程的入口，我们来看一下ThreadUtil.execute的具体实现

``` java
  /**
   * Parallel execution of the <code>tasks</code>. The startup is synchronized so this method
   * emulates a load test.
   * @param tasks the list of tasks to be run
   * @param threadPoolSize the size of the parallel threads to be used to execute the tasks
   * @param timeout a maximum timeout to wait for tasks finalization
   * @param triggerAtOnce <tt>true</tt> if the parallel execution of tasks should be trigger at once
   */
  public static final void execute(List<? extends Runnable> tasks, int threadPoolSize,
      long timeout, boolean triggerAtOnce) {
    final CountDownLatch startGate= new CountDownLatch(1);
    final CountDownLatch endGate= new CountDownLatch(tasks.size());

    Utils.log("ThreadUtil", 2, "Starting executor timeOut:" + timeout + "ms"
        + " workers:" + tasks.size() + " threadPoolSize:" + threadPoolSize);
    ExecutorService pooledExecutor = // Executors.newFixedThreadPool(threadPoolSize);
        new ThreadPoolExecutor(threadPoolSize, threadPoolSize,
        timeout, TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<Runnable>(),
        new ThreadFactory() {
          @Override
          public Thread newThread(Runnable r) {
            Thread result = new Thread(r);
            result.setName(THREAD_NAME);
            return result;
          }
        });

    List<Callable<Object>> callables = Lists.newArrayList();
    for (final Runnable task : tasks) {
      callables.add(new Callable<Object>() {

        @Override
        public Object call() throws Exception {
          task.run();
          return null;
        }

      });
    }
    try {
      if (timeout != 0) {
        pooledExecutor.invokeAll(callables, timeout, TimeUnit.MILLISECONDS);
      } else {
        pooledExecutor.invokeAll(callables);
      }
    } catch (InterruptedException handled) {
      handled.printStackTrace();
      Thread.currentThread().interrupt();
    } finally {
      pooledExecutor.shutdown();
    }
  }
```
问题就出现了，ThreadUtil.execute(m_workers, 1, 10 * 1000L, false);的调用中设定了线程池的大小为1，而且maxpoolsize = corepoolsize = 1 ,这个问题严重了，所谓的多线程其实是个单线程，这样永远只有一个机器可以执行任务。其实我们只要这样修改就好了

``` java
public void awaitTermination(long timeout) throws InterruptedException
	{
		ThreadUtil.execute(m_workers, m_hosts.length(), 10 * 1000L, false);
	}
```

把线程池的大小修改成slave的机器的数量就，这样就解决了线程池的问题。其次，invokeAll这个方法是非阻塞的，这个会导致Master进程会退出，所以的子线程会被丢失，这样的结果一旦子线程完成当前任务，尝试再次连接我们的master服务就会发生问题，从而发生雪崩效应，导致整个分布式集群的crash，看一下下面的slave进程连接master的代码就知道了，这个是个死循环啊！！！
``` java
public void waitForSuites() {
		try {
			while (true) {
				//TODO set timeout
				XmlSuite s = m_slaveAdpter.getSuite(Long.MAX_VALUE);
				if( s== null) {
          continue;
        }
				log("Processing " + s.getName());
				List<XmlSuite> suites = Lists.newArrayList();
				suites.add(s);
				m_testng.setXmlSuites(suites);
				List<ISuite> suiteRunners = m_testng.runSuitesLocally();
				ISuite sr = suiteRunners.get(0);
				log("Done processing " + s.getName());
				m_slaveAdpter.returnResult(sr);
			}
		}
		catch(Exception ex) {
			ex.printStackTrace(System.out);
		}
	}
```

那么怎么办呢，我们知道master其实是把所有的runnable转换成了callable了，那么我们只要遍列所有的callable对象，尝试获取一下结果就好了，因为这里会block我们的master进程，知道最后的退出。

3. 序列化问题(有待解决)

sokcet通信里面一个问题是序列化，TestNG的序列化这块非常多的问题，所有的Slave在完成任务以后，会把一个SuiteRunner的对象返回回来，这个SuiteRunner对象可是非常的复杂，深度嵌套而且隐患无数，我们先来看一下下面的代码

``` java
public class SuiteRunner implements ISuite, Serializable, IInvokedMethodListener {

  /* generated */
  private static final long serialVersionUID = 5284208932089503131L;

  private static final String DEFAULT_OUTPUT_DIR = "test-output";

  private Map<String, ISuiteResult> m_suiteResults = Collections.synchronizedMap(Maps.<String, ISuiteResult>newLinkedHashMap());
  transient private List<TestRunner> m_testRunners = Lists.newArrayList();
  transient private List<ISuiteListener> m_listeners = Lists.newArrayList();
  transient private TestListenerAdapter m_textReporter = new TestListenerAdapter();

  private String m_outputDir; // DEFAULT_OUTPUT_DIR;
  private XmlSuite m_suite;
  private Injector m_parentInjector;

  transient private List<ITestListener> m_testListeners = Lists.newArrayList();
  transient private ITestRunnerFactory m_tmpRunnerFactory;

  transient private ITestRunnerFactory m_runnerFactory;
  transient private boolean m_useDefaultListeners = true;

  // The remote host where this suite was run, or null if run locally
  private String m_host;

  // The configuration
  transient private IConfiguration m_configuration;

  transient private ITestObjectFactory m_objectFactory;
  transient private Boolean m_skipFailedInvocationCounts = Boolean.FALSE;

  transient private IMethodInterceptor m_methodInterceptor;
  private List<IInvokedMethodListener> m_invokedMethodListeners;

  /** The list of all the methods invoked during this run */
  private List<IInvokedMethod> m_invokedMethods =
      Collections.synchronizedList(Lists.<IInvokedMethod>newArrayList());

  private List<ITestNGMethod> m_allTestMethods = Lists.newArrayList();
}
```

这个问题复杂了，我们要么重写整个返回结构，否则我们很可能由于使用的DataProvider的参数不能序列化而导致通信的失败。目前我们采用的方式是完全不返回SuiteRunner ，只是返回一个简单的枚举OK,FAIL,然后通过每个Slave把结果输出到一个共享目录，最终再整合。

今天只是简单的把之前研究TestNG的结果分享在这里，如果你有兴趣觉得我写的东西对你有用，请和我一起祈祷上苍，保佑我奶奶身体健康，长命百岁！！！

谢谢。
