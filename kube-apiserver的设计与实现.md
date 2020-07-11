# 3 Flink RPC

对于Flink中各个组件（JobMaster、TaskManager、Dispatcher等），其底层RPC框架基于Akka实现，本文着重分析Flink中的Rpc框架实现机制及梳理其通信流程。

## 3.1 Akka介绍

Akka是一个开发并发、容错和可伸缩应用的框架。它是Actor Model的一个实现，和Erlang的并发模型很像。在Actor模型中，所有的实体被认为是独立的actors。actors和其他actors通过发送异步消息通信。Actor模型的强大来自于异步。它也可以显式等待响应，这使得可以执行同步操作。但是，强烈不建议同步消息，因为它们限制了系统的伸缩性。每个actor有一个邮箱(mailbox)，它收到的消息存储在里面。另外，每一个actor维护自身单独的状态。一个Actors网络如下所示：

![avatar](picture/akka-actors.png)

每个actor是一个单一的线程，它不断地从其邮箱中poll(拉取)消息，并且连续不断地处理。对于已经处理过的消息的结果，actor可以改变它自身的内部状态或者发送一个新消息或者孵化一个新的actor。尽管单个的actor是自然有序的，但一个包含若干个actor的系统却是高度并发的并且极具扩展性的。因为那些处理线程是所有actor之间共享的。这也是我们为什么不该在actor线程里调用可能导致阻塞的“调用”。因为这样的调用可能会阻塞该线程使得他们无法替其他actor处理消息。

## 3.2 RPC类图结构

下图展示了Flink中RPC框架中涉及的主要类。

![avatar](picture/FlinkRPC类图结构.png)

### 3.2.1 RpcGateway

Flink的RPC协议通过RpcGateway来定义；由前面可知，若想与远端Actor通信，则必须提供地址（ip和port），如在Flink-on-Yarn模式下，JobMaster会先启动ActorSystem，此时TaskExecutor的Container还未分配，后面与TaskExecutor通信时，必须让其提供对应地址，从类继承图可以看到基本上所有组件都实现了RpcGateway接口，其代码如下：

RpcGateway 接口是用于远程调用的代理接口。 RpcGateway 提供了获取其所代理的 RpcEndpoint 的地址的方法。在实现一个提供 RPC 调用的组件时，通常需要先定一个接口，该接口继承 RpcGateway 并约定好提供的远程调用的方法。

```
public interface RpcGateway {

	/**
	 * Returns the fully qualified address under which the associated rpc endpoint is reachable.
	 *
	 * @return Fully qualified (RPC) address under which the associated rpc endpoint is reachable
	 */
	String getAddress();

	/**
	 * Returns the fully qualified hostname under which the associated rpc endpoint is reachable.
	 *
	 * @return Fully qualified hostname under which the associated rpc endpoint is reachable
	 */
	String getHostname();
}
```

### 3.2.2 RpcEndpoint

每个RpcEndpoint对应了一个路径（endpointId和actorSystem共同确定），每个路径对应一个Actor，其实现了RpcGateway接口，其构造函数如下：

RpcEndpoint 是对 RPC 框架中提供具体服务的实体的抽象，所有提供远程调用方法的组件都需要继承该抽象类。另外，对于同一个 RpcEndpoint 的所有 RPC 调用都会在同一个线程（RpcEndpoint 的“主线程”）中执行，因此无需担心并发执行的线程安全问题。

```
protected RpcEndpoint(final RpcService rpcService, final String endpointId) {
	// 保存rpcService和endpointId
	this.rpcService = checkNotNull(rpcService, "rpcService");
	this.endpointId = checkNotNull(endpointId, "endpointId");
	// 通过RpcService启动RpcServer
	this.rpcServer = rpcService.startServer(this);
	// 主线程执行器，所有调用在主线程中串行执行
	this.mainThreadExecutor = new MainThreadExecutor(rpcServer, 	this::validateRunsInMainThread);
}
```

在RpcEndpoint中还定义了一些方法如runAsync(Runnable)、callAsync(Callable, Time)方法来执行Rpc调用，值得注意的是在Flink的设计中，对于同一个Endpoint，所有的调用都运行在主线程，因此不会有并发问题，当启动RpcEndpoint/进行Rpc调用时，其会委托RcpServer进行处理。

### 3.2.3 RpcService

RpcService 是 RpcEndpoint 的运行时环境， RpcService 提供了启动 RpcEndpoint, 连接到远端 RpcEndpoint 并返回远端 RpcEndpoint 的代理对象等方法。此外， RpcService 还提供了某些异步任务或者周期性调度任务的方法。

Rpc服务的接口，其主要作用如下：

- 根据提供的RpcEndpoint来启动RpcServer（Actor）；
- 根据提供的地址连接到RpcServer，并返回一个RpcGateway；
- 延迟/立刻调度Runnable、Callable；
- 停止RpcServer（Actor）或自身服务；

在Flink中其实现类为AkkaRpcService

### 3.2.4 其他组件

RpcServer 相当于 RpcEndpoint 自身的的代理对象（self gateway)。RpcServer 是 RpcService 在启动了 RpcEndpoint 之后返回的对象，每一个 RpcEndpoint 对象内部都有一个 RpcServer 的成员变量，通过 getSelfGateway 方法就可以获得自身的代理，然后调用该Endpoint 提供的服务。

FencedRpcEndpoint 和 FencedRpcGate 要求在调用 RPC 方法时携带 token 信息，只有当调用方提供了 token 和 endpoint 的 token 一致时才允许调用。

## 3.2 基于 Akka 的 RPC 实现

前面介绍了 Flink 内部 RPC 框架的基本抽象，主要就是 RpcService, RpcEndpoint, RpcGateway, RpcServer 等接口。至于具体的实现，则可以有多种不同的方式，如 Akka， Netty 等。Flink 目前提供了一套基于 Akka 的实现。

### 3.2.1 启动 RpcEndpoint

AkkaRpcService 实现了 RpcService 接口， AkkaRpcService 会启动 Akka actor 来接收来自 RpcGateway 的 RPC 调用。

首先，在 RpcEndpoint 的构造函数中，会调用 AkkaRpcService 的 startServer 方法来初始化服务，

```
protected RpcEndpoint(final RpcService rpcService, final String endpointId) {
	this.rpcService = checkNotNull(rpcService, "rpcService");
	this.endpointId = checkNotNull(endpointId, "endpointId");

	this.rpcServer = rpcService.startServer(this);

	this.mainThreadExecutor = new MainThreadExecutor(rpcServer, this::validateRunsInMainThread);
}
```

AkkaRpcService 的 startServer 方法主要工作包括：

- 调用registerAkkaRpcActor中的registerAkkaRpcActor创建一个 AkkaRpcActor 或 FencedAkkaRpcActor
- 通过动态代理创建代理对象

```
public <C extends RpcEndpoint & RpcGateway> RpcServer startServer(C rpcEndpoint) {
	checkNotNull(rpcEndpoint, "rpc endpoint");

	final SupervisorActor.ActorRegistration actorRegistration = registerAkkaRpcActor(rpcEndpoint);
	final ActorRef actorRef = actorRegistration.getActorRef();
	final CompletableFuture<Void> actorTerminationFuture = actorRegistration.getTerminationFuture();

	LOG.info("Starting RPC endpoint for {} at {} .", rpcEndpoint.getClass().getName(), actorRef.path());

	final String akkaAddress = AkkaUtils.getAkkaURL(actorSystem, actorRef);
	final String hostname;
	Option<String> host = actorRef.path().address().host();
	if (host.isEmpty()) {
		hostname = "localhost";
	} else {
		hostname = host.get();
	}

	Set<Class<?>> implementedRpcGateways = new HashSet<>(RpcUtils.extractImplementedRpcGateways(rpcEndpoint.getClass()));

	implementedRpcGateways.add(RpcServer.class);
	implementedRpcGateways.add(AkkaBasedEndpoint.class);

	final InvocationHandler akkaInvocationHandler;

	if (rpcEndpoint instanceof FencedRpcEndpoint) {
		// a FencedRpcEndpoint needs a FencedAkkaInvocationHandler
		akkaInvocationHandler = new FencedAkkaInvocationHandler<>(
			akkaAddress,
			hostname,
			actorRef,
			configuration.getTimeout(),
			configuration.getMaximumFramesize(),
			actorTerminationFuture,
			((FencedRpcEndpoint<?>) rpcEndpoint)::getFencingToken,
			captureAskCallstacks);

		implementedRpcGateways.add(FencedMainThreadExecutable.class);
	} else {
		akkaInvocationHandler = new AkkaInvocationHandler(
			akkaAddress,
			hostname,
			actorRef,
			configuration.getTimeout(),
			configuration.getMaximumFramesize(),
			actorTerminationFuture,
			captureAskCallstacks);
	}

	// Rather than using the System ClassLoader directly, we derive the ClassLoader
	// from this class . That works better in cases where Flink runs embedded and all Flink
	// code is loaded dynamically (for example from an OSGI bundle) through a custom ClassLoader
	ClassLoader classLoader = getClass().getClassLoader();

	@SuppressWarnings("unchecked")
	RpcServer server = (RpcServer) Proxy.newProxyInstance(
		classLoader,
		implementedRpcGateways.toArray(new Class<?>[implementedRpcGateways.size()]),
		akkaInvocationHandler);

	return server;
}
```

```
private <C extends RpcEndpoint & RpcGateway> SupervisorActor.ActorRegistration registerAkkaRpcActor(C rpcEndpoint) {
	final Class<? extends AbstractActor> akkaRpcActorType;

	if (rpcEndpoint instanceof FencedRpcEndpoint) {
		akkaRpcActorType = FencedAkkaRpcActor.class;
	} else {
		akkaRpcActorType = AkkaRpcActor.class;
	}

	synchronized (lock) {
		checkState(!stopped, "RpcService is stopped");

		final SupervisorActor.StartAkkaRpcActorResponse startAkkaRpcActorResponse = SupervisorActor.startAkkaRpcActor(
			supervisor.getActor(),
			actorTerminationFuture -> Props.create(
				akkaRpcActorType,
				rpcEndpoint,
				actorTerminationFuture,
				getVersion(),
				configuration.getMaximumFramesize()),
			rpcEndpoint.getEndpointId());

		final SupervisorActor.ActorRegistration actorRegistration = startAkkaRpcActorResponse.orElseThrow(cause -> new AkkaRpcRuntimeException(
			String.format("Could not create the %s for %s.",
				AkkaRpcActor.class.getSimpleName(),
				rpcEndpoint.getEndpointId()),
			cause));

		actors.put(actorRegistration.getActorRef(), rpcEndpoint);

		return actorRegistration;
	}
}
```

在 RpcEndpoint 对象创建后，下一步操作是调用start方法启动它

```
public final void start() {
		rpcServer.start();
	}
```

实际上调用的是 RpcServer.start() 方法。RpcServer 是通过 AkkaInvocationHandler 创建的动态代理对象：

```
public void start() {
	rpcEndpoint.tell(ControlMessages.START, ActorRef.noSender());
}
```

所以启动 RpcEndpoint 实际上就是向当前 endpoint 绑定的 Actor 发送一条 START 消息，通知服务启动。

### 3.2.2 获取 RpcEndpoint 的代理对象

在 RpcEndpoint 创建的过程中，实际上已经通过动态代理生成了一个可供本地使用的代理对象，通过 RpcEndpoint 的 getSelfGateway 方法可以直接获取。

```
public <C extends RpcGateway> C getSelfGateway(Class<C> selfGatewayType) {
	if (selfGatewayType.isInstance(rpcServer)) {
		@SuppressWarnings("unchecked")
		C selfGateway = ((C) rpcServer);

		return selfGateway;
	} else {
		throw new RuntimeException("RpcEndpoint does not implement the RpcGateway interface of type " + selfGatewayType + '.');
	}
}
```

如果需要获取一个远程 RpcEndpoint 的代理，就需要通过 RpcService 的 connect 方法，需要提供远程 endpoint 的地址：
实现RpcService的是AkkaRpcService

方法主要的功能包括：

通过地址获取 RpcEndpoint 绑定的 actor 的引用 ActorRef
向对应的 AkkaRpcActor 发送握手消息
握手成功之后，创建 AkkaInvocationHandler 对象，并通过动态代理生成代理对象

```
public <C extends RpcGateway> CompletableFuture<C> connect(
		final String address,
		final Class<C> clazz) {

	return connectInternal(
		address,
		clazz,
		(ActorRef actorRef) -> {
			Tuple2<String, String> addressHostname = extractAddressHostname(actorRef);

			return new AkkaInvocationHandler(
				addressHostname.f0,
				addressHostname.f1,
				actorRef,
				configuration.getTimeout(),
				configuration.getMaximumFramesize(),
				null,
				captureAskCallstacks);
		});
}
```

```
private <C extends RpcGateway> CompletableFuture<C> connectInternal(
		final String address,
		final Class<C> clazz,
		Function<ActorRef, InvocationHandler> invocationHandlerFactory) {
	checkState(!stopped, "RpcService is stopped");

	LOG.debug("Try to connect to remote RPC endpoint with address {}. Returning a {} gateway.",
		address, clazz.getName());

	final CompletableFuture<ActorRef> actorRefFuture = resolveActorAddress(address);

	final CompletableFuture<HandshakeSuccessMessage> handshakeFuture = actorRefFuture.thenCompose(
		(ActorRef actorRef) -> FutureUtils.toJava(
			Patterns
				.ask(actorRef, new RemoteHandshakeMessage(clazz, getVersion()), configuration.getTimeout().toMilliseconds())
				.<HandshakeSuccessMessage>mapTo(ClassTag$.MODULE$.<HandshakeSuccessMessage>apply(HandshakeSuccessMessage.class))));

	return actorRefFuture.thenCombineAsync(
		handshakeFuture,
		(ActorRef actorRef, HandshakeSuccessMessage ignored) -> {
			InvocationHandler invocationHandler = invocationHandlerFactory.apply(actorRef);

			// Rather than using the System ClassLoader directly, we derive the ClassLoader
			// from this class . That works better in cases where Flink runs embedded and all Flink
			// code is loaded dynamically (for example from an OSGI bundle) through a custom ClassLoader
			ClassLoader classLoader = getClass().getClassLoader();

			@SuppressWarnings("unchecked")
			C proxy = (C) Proxy.newProxyInstance(
				classLoader,
				new Class<?>[]{clazz},
				invocationHandler);

			return proxy;
		},
		actorSystem.dispatcher());
}
```

### 3.2.3 Rpc 调用

在获取了本地或者远端 RpcEndpoint 的代理对象后，就可以通过代理对象发起 RPC 调用了。由于代理对象是通过动态代理创建的，因而所以的方法都会转化为 AkkaInvocationHandler 的 invoke 方法，并传入 RPC 调用的方法以及参数信息。

```
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	Class<?> declaringClass = method.getDeclaringClass();

	Object result;

	if (declaringClass.equals(AkkaBasedEndpoint.class) ||
		declaringClass.equals(Object.class) ||
		declaringClass.equals(RpcGateway.class) ||
		declaringClass.equals(StartStoppable.class) ||
		declaringClass.equals(MainThreadExecutable.class) ||
		declaringClass.equals(RpcServer.class)) {
		result = method.invoke(this, args);
	} else if (declaringClass.equals(FencedRpcGateway.class)) {
		throw new UnsupportedOperationException("AkkaInvocationHandler does not support the call FencedRpcGateway#" +
			method.getName() + ". This indicates that you retrieved a FencedRpcGateway without specifying a " +
			"fencing token. Please use RpcService#connect(RpcService, F, Time) with F being the fencing token to " +
			"retrieve a properly FencedRpcGateway.");
	} else {
		result = invokeRpc(method, args);
	}

	return result;
}
```

对于 RPC 调用，需要将 RPC 调用的方法名、参数类型和参数值封装为一个 RpcInvocation 对象，根据 RpcEndpoint 是本地的还是远端，具体的 有 LocalRpcInvocation 和 RemoteRpcInvocation 两类，它们的区别在于是否需要序列化。

然后根据 RPC 方法是否有返回值，决定调用 tell 或 ask 方法，然后通过 Akka 的 ActorRef 向对应的 AkkaRpcActor 发送请求，如果带有返回值，则等待 actor 的响应。

### 3.2.4 AkkaRpcActor

AkkaRpcActor 负责接受 RPC 调用的请求，并通过反射调用 RpcEndpoint 的对应方法来完成 RPC 调用。

首先，通过 AkkaRpcActor 的 createReceive 方法来接受HandshakeMessage，handleControlMessage，还有handleMessage

```
public Receive createReceive() {
	return ReceiveBuilder.create()
		.match(RemoteHandshakeMessage.class, this::handleHandshakeMessage)
		.match(ControlMessages.class, this::handleControlMessage)
		.matchAny(this::handleMessage)
		.build();
}
```

然后通过handleMessage处理RPC调用

```
private void handleMessage(final Object message) {
	if (state.isRunning()) {
		mainThreadValidator.enterMainThread();

		try {
			handleRpcMessage(message);
		} finally {
			mainThreadValidator.exitMainThread();
		}
	} else {
		log.info("The rpc endpoint {} has not been started yet. Discarding message {} until processing is started.",
			rpcEndpoint.getClass().getName(),
			message.getClass().getName());

		sendErrorIfSender(new AkkaRpcException(
			String.format("Discard message, because the rpc endpoint %s has not been started yet.", rpcEndpoint.getAddress())));
	}
}
```

可以看到核心方法是handleRpcMessage，分成handleRunAsync，handleCallAsync 和 handleRpcInvocation，
如果发来的Message是 RunAsync，交给 handleRunAsync 方法，
如果发来的Message是 CallAsync，交给 handleCallAsync 方法，
handleRpcInvocation处理远程的RPC调用


```
protected void handleRpcMessage(Object message) {
	if (message instanceof RunAsync) {
		handleRunAsync((RunAsync) message);
	} else if (message instanceof CallAsync) {
		handleCallAsync((CallAsync) message);
	} else if (message instanceof RpcInvocation) {
		handleRpcInvocation((RpcInvocation) message);
	} else {
		log.warn(
			"Received message of unknown type {} with value {}. Dropping this message!",
			message.getClass().getName(),
			message);

		sendErrorIfSender(new AkkaUnknownMessageException("Received unknown message " + message +
			" of type " + message.getClass().getSimpleName() + '.'));
	}
}
```

```
private void handleRpcInvocation(RpcInvocation rpcInvocation) {
		Method rpcMethod = null;

		try {
			String methodName = rpcInvocation.getMethodName();
			Class<?>[] parameterTypes = rpcInvocation.getParameterTypes();

			rpcMethod = lookupRpcMethod(methodName, parameterTypes);
		} catch (ClassNotFoundException e) {
			log.error("Could not load method arguments.", e);

			RpcConnectionException rpcException = new RpcConnectionException("Could not load method arguments.", e);
			getSender().tell(new Status.Failure(rpcException), getSelf());
		} catch (IOException e) {
			log.error("Could not deserialize rpc invocation message.", e);

			RpcConnectionException rpcException = new RpcConnectionException("Could not deserialize rpc invocation message.", e);
			getSender().tell(new Status.Failure(rpcException), getSelf());
		} catch (final NoSuchMethodException e) {
			log.error("Could not find rpc method for rpc invocation.", e);

			RpcConnectionException rpcException = new RpcConnectionException("Could not find rpc method for rpc invocation.", e);
			getSender().tell(new Status.Failure(rpcException), getSelf());
		}

		if (rpcMethod != null) {
			try {
				// this supports declaration of anonymous classes
				rpcMethod.setAccessible(true);

				if (rpcMethod.getReturnType().equals(Void.TYPE)) {
					// No return value to send back
					rpcMethod.invoke(rpcEndpoint, rpcInvocation.getArgs());
				}
				else {
					final Object result;
					try {
						result = rpcMethod.invoke(rpcEndpoint, rpcInvocation.getArgs());
					}
					catch (InvocationTargetException e) {
						log.debug("Reporting back error thrown in remote procedure {}", rpcMethod, e);

						// tell the sender about the failure
						getSender().tell(new Status.Failure(e.getTargetException()), getSelf());
						return;
					}

					final String methodName = rpcMethod.getName();

					if (result instanceof CompletableFuture) {
						final CompletableFuture<?> responseFuture = (CompletableFuture<?>) result;
						sendAsyncResponse(responseFuture, methodName);
					} else {
						sendSyncResponse(result, methodName);
					}
				}
			} catch (Throwable e) {
				log.error("Error while executing remote procedure call {}.", rpcMethod, e);
				// tell the sender about the failure
				getSender().tell(new Status.Failure(e), getSelf());
			}
		}
	}
```

## 3.3 小结

这篇文章简单地分析了 Flink 内部的 RPC 框架。首先，通过 RpcService, RpcEndpoint, RpcGateway, RpcServer 等接口和抽象类，确定了 RPC 服务的基本框架；在这套框架的基础上， Flink 借助 Akka 和动态代理等技术提供了 RPC 调用的具体实现。
