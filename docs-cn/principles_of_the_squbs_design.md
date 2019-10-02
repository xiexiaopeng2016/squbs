# squbs的设计原则

squbs平台的设计围绕以下设计原则：

1. **关注点分离**

	 应用程序代码应专注于应用程序和/或业务逻辑。生命周期和基础架构问题应留给基础架构。基础结构只需要来自应用程序的输入，而这些输入不能自动派发。squbs的设计正是这样做的：允许基础结构插入基础结构问题，而不会以任何方式干扰应用程序。

2. **松耦合**
	 
	 大型系统的混乱来自类型系统的紧密耦合。作为消息传递系统，Akka可以完全避免类型耦合，允许模块共存，而且彼此之间没有任何类型依赖关系。但是，启动参与者的常用方法需要参与者之间具有类型耦合
   However, the common way of starting actors needs actors to have type coupling between each others. For instance, `context.actorOf(Props[MyActorType])` requires the caller to import MyActorType and thus causes a type dependency to `MyActorType`. Such couplings cannot be categorized as loose coupling. squbs allows creation of loosely-coupled components called "cubes" that may only communicate by messages known to both parties. To what degree such cubes should be used depends on the complexity of the service, and whether the functionality provided by a cube is optional or not. A very small "microservice" may be all implemented in one cube. A more complex service that has common modules shared with other services may want to implement that common logic as a separate cube (much more than a separate library as that is usually tightly-coupled). An admin console that may be deployed with all services but rarely communicates with the services directly should definitely be implemented as a separate cube. 
   
3. **DRY**  

   Don't Repeat Yourself. The drop-in design of squbs allows for at-most-once coupling. This generally means cubes can be dropped in by just adding them to the dependency list or runtime classpath. No real call into the cube is required as this will cause dependencies both in the code and in the dependency list/classpath. Removing the loosely-coupled module will therefore be as easy as removing it from the runtime classpath or dependency.
   
4. **Minimize Duplication of Effort**

   Most of the low-level components of the application that deal with application lifecycle and monitoring are common across applications. A minimal framework dealing with the lifecycle management will hide and abstract away such duplications. This will standardize the PaaS and load balancer interfaces and allows developers to focus on their concern (application logic) and dramatically reduces the variation of scripts and configuration needed to support the applications.
   
5. **Thin Platform**

   We realize the emergence of the thin platform and try to keep squbs as lightweight and non-imposing as possible. As a result, we have squbs-unicomplex which is the core container in charge of bootstrapping and managing the state and lifecycle of the runtime. It pulls in virtually no dependency beyond Scala and Akka and does not impose further requirements or frameworks on the developer. Any other framework can be included by the developer as needed.

  
## Common Questions/Concerns from Akka Developers

**Developer:** I want to start from my own main

**squbs:** How do you wish to start supporting infrastructure such as admin consoles? Mixing in creates a compile-time dependency as well as developer awareness and violates both separation of concerns and loose coupling.

**Developer:** I will call the initializers for all the infrastructure

**squbs:** So you need both the dependency in your build definition as well as modifying the application to call into initialization code? This violates separation of concerns, loose coupling, and DRY.

**Developer:** We can provide lifecycle scripts how to start our application.

**squbs:** This means we need a separate lifecycle scripts for each application. This would be duplication of effort to have different scripts for each application?

**Developer:** We need to write our own health check page.

**squbs:** Having standardized admin cube allows all squbs applications to be viewed similarly.

**Developer:** We would need to configure PaaS and Load Balancer for the application.

**squbs:** We should not need to have separate configurations for each application. We should not maintain these many configurations, one per application. This is duplication of effort. We would not achieve separation of concern with this. Should we not standardize?

**squbs:** How should a unified cloud management/PaaS environment cleanly shut down the application such as at flex-up or flex-down?

**Developer:** Call my shutdown script

**squbs:** How do you make sure the shutdown script gracefully terminates the process without causing any state and data corruption?

**Developer:** We take care of graceful shutdown of the application.

**squbs:** So each application will have to take care of this themselves. Shutdown is not trivial. What if there are bugs in the shutdown process. This is best handled by standard scripts, separate from the application and will avoid both duplication of efforts as well as separation of concerns.

