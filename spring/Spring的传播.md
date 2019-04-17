# Spring的传播
Spring中七种事务传播行为
事务传播行为类型	说明

- PROPAGATION_REQUIRED	如果当前没有事务，就新建一个事务，如果已经存在一个事务中，加入到这个事务中。这是最常见的选择。
- PROPAGATION_SUPPORTS	支持当前事务，如果当前没有事务，就以非事务方式执行。
- PROPAGATION_MANDATORY	使用当前的事务，如果当前没有事务，就抛出异常。
- PROPAGATION_REQUIRES_NEW	新建事务，如果当前存在事务，把当前事务挂起。
- PROPAGATION_NOT_SUPPORTED	以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。
- PROPAGATION_NEVER	以非事务方式执行，如果当前存在事务，则抛出异常。
- PROPAGATION_NESTED	如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与- PROPAGATION_REQUIRED类似的操作。



其实就是一个事务和嵌套事务的关系


不管是那种隔离等级，如果一个方法中调用了两个事务方法，外围方法未添加事务，则内部的两个事务相互隔离。独立回滚

## Propagation.REQUIRED
如果外围方法添加了事务，则内部的事务看成一个整体。即使一个异常被catch，没有被外围捕获，也需要都回滚


## PROPAGATION_REQUIRES_NEW


