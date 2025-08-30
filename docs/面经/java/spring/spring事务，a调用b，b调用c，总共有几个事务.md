在Spring事务中，当方法a调用b，b再调用c时，总共有几个事务并不是一个固定的数字，而是取决于每个方法上@Transactional注解的事务传播机制（Transaction Propagation）配置。

事务传播机制定义了当一个事务方法被另一个事务方法调用时，事务应该如何传播。 以下是几种常见的场景：

### 场景一：全部使用默认配置 (最常见)
如果方法a、b、c都使用了@Transactional注解，并且没有指定传播机制，那么它们都将使用默认的`Propagation.REQUIRED`。

*   `Propagation.REQUIRED`: 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。

在这种情况下：
1.  外部调用方法a时，Spring会检查当前是否存在事务。假设没有，就会为a创建一个新的事务。
2.  方法a在执行过程中调用方法b。Spring检查到b的传播机制是`REQUIRED`，并且当前已经存在一个由a创建的事务，所以b会加入到a的事务中。
3.  方法b在执行过程中调用方法c。同样，c也会加入到a的事务中。

结论：在这种最常见的情况下，整个调用链中只有一个事务。如果任何一个方法（a、b或c）抛出异常，整个事务将回滚。

### 场景二：创建新事务
如果某个方法的传播机制被设置为`Propagation.REQUIRES_NEW`，情况就会不同。

*   `Propagation.REQUIRES_NEW`: 每次都会创建一个新的事务。如果当前已经存在一个事务，那么会把当前事务挂起。

让我们看一个例子：
*   方法a: `@Transactional(propagation = Propagation.REQUIRED)`
*   方法b: `@Transactional(propagation = Propagation.REQUIRES_NEW)`
*   方法c: `@Transactional(propagation = Propagation.REQUIRED)`

执行流程如下：
1.  外部调用a，a创建了一个事务（我们称之为事务A）。
2.  a调用b。由于b的传播机制是`REQUIRES_NEW`，Spring会挂起事务A，并为b创建一个全新的事务（我们称之为事务B）。
3.  b调用c。由于c的传播机制是`REQUIRED`，它会加入当前正在执行的事务，也就是事务B。
4.  c执行完毕，然后b执行完毕。事务B会首先提交。
5.  之后，挂起的事务A会恢复，继续执行a方法中剩余的逻辑，最后事务A提交。

结论：在这种情况下，总共会创建两个事务（事务A和事务B）。事务B的成功或失败不会影响事务A（除非b方法将异常继续向上抛出给了a）。

### 场景三：每个方法都创建新事务
如果a、b、c的传播机制都配置为`Propagation.REQUIRES_NEW`，那么：
1.  a调用时，创建事务A。
2.  a调用b时，挂起事务A，创建新的事务B。
3.  b调用c时，挂起事务B，创建新的事务C。

结论：在这种情况下，总共会创建三个独立的事务。

### 其他传播机制
除了`REQUIRED`和`REQUIRES_NEW`，Spring还提供了其他几种传播机制，如：
*   `NESTED`: 如果当前存在事务，则在嵌套事务内执行。嵌套事务是外部事务的一部分，但可以独立于外部事务进行回滚。
*   `SUPPORTS`: 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
*   `NOT_SUPPORTED`: 以非事务方式执行操作。如果当前存在事务，则把它挂起。

### 重要提醒：同类中的方法调用
一个常见的陷阱是，如果方法a、b、c在同一个类中，并且方法a没有被@Transactional注解，那么直接通过`this`关键字从a调用b和c，即使b和c有@Transactional注解，事务也不会生效。这是因为Spring的事务管理是基于代理实现的，内部方法调用会绕过代理，从而导致事务注解失效。