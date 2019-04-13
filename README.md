
使用指南1.1.x:https://github.com/changmingxie/tcc-transaction/wiki/%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%971.1.x

1.1.x 源码分支：https://github.com/changmingxie/tcc-transaction/tree/master

使用指南1.2.x:https://github.com/changmingxie/tcc-transaction/wiki/%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%971.2.x

1.2.x 源码分支：https://github.com/changmingxie/tcc-transaction/tree/master-1.2.x

1.2.x 版本不向下兼容1.1.x，主要在声明tcc服务方法的注解有改变。1.2.x不同于1.1.x主要的地方在于发布服务时不再强制要求服务方法参数必须有TransactionContext参数，从而减少对业务代码的侵入。


入口为ResourceCoordinatorAspect 和 CompensableTransactionAspect这两个springaop拦截器，拦截@Compensable注解已获取事务信息，并进行配置、持久化。ResourceCoordinatorAspect优先级更高，先执行；CompensableTransactionAspect优先级次之，后执行；最后执行被代理对象的目标方法。

@Compensable注解出现在服务调用方（ROOT）和被调用方（PROVIDER）的try方法上。被调用方要在提供的接口client方法中（类似dubbo的domain包）加入@Compensable注解，提供到调用方工程，用于被拦截已加入到现有事务（Transaction）的Participant列表中。也就是服务调用方调用的try方法，没调用一个外部服务的try方法，都会拦截到@Compensable，加入主事务，用于记录，补偿。

拦截到@Compensable后先执行ResourceCoordinatorAspect中的处理，该aspect会检查全局TransactionManager中的事务队列（ThreadLocal中的队列）是否有事务存在，若没有，表示为服务调用方新开启的事务，则不做任何处理；若当前线程的事务队列中存在事务（Transaction），则说明代码执行到了当前工程try方法中的一个外部的try方法，判断其事务状态（TRYING,CONFIRMING,CANCELLING）。状态为TRYING,则将这个外部try方法的@Compensable注解中的信息取出，包括method信息（反射）tcc事务中规定的confirmMethodName，cancelMethodName，新加事务xid，xid的golbalid同主事务id，branchid为新建（在db总不同brunch为一条记录，但都被关联在一个global），封装成Participant对象，加入到主事务（Transaction）中的participants列表；CONFIRMING状态和CANCELLING状态不做操作。

之后逻辑由CompensableTransactionAspect接管。CompensableTransactionAspect首先根据ResourceCoordinatorAspect封装好的参数和@Compensable及其修饰的方法中TransactionContext的信息、事务激活状态（线程本地事务队列是否有值）、事务传递方式分析事务发生地点（ROOT、PROVIDER、NORMAL）。
若地点发生在事务发起方且之前无事务（ROOT），则transactionManager开启新事务信息见Transaction对象，持久化，并注册到线程本地事务队列，执行目标方法pjp.proceed()，若执行成功commit，若捕获到异常rollback。
若地点发生在事务发起方调用的远程服务内部（PROVIDER），先判断传递过来的事务状态（TRYING,CONFIRMING,CANCELLING）。若为TRYING，在将transactionContext传递过来的发起方的事务信息持久化到远程工程内部的环境（持久化，注册到远程工程线程内部的事务队列中），xid设置为服务发起方的xid，分支类型设置为BRANCH，事务状态TRYING，执行目标方法pjp.proceed()；若事务状态为CONFIRMING，根据事务xid找到在持久化中找到事务Transaction对象，执行事务commit操作；若为CANCELLING，在持久化中找到事务Transaction对象执行rollback操作。之后清理本地线程队列中的事务信息
若发生在主事务调动外部接口时（NORMAL），前面的ResourceCoordinatorAspect拦截器已经进行过加入到本地主事务的处理，无需额外操作，直接执行目标方法jp.proceed()。
