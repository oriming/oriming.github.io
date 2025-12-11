---
title: Spring 事务
author: Ori Ming
date: 2023-10-04 11:13:00 +0800
categories: [Spring, 事务]
tags: [Spring]
render_with_liquid: false
---

# Welcome

本文主要做笔记，加深对 Spring 事务的理解。内容主要有三点：

+ 源码原理
+ 传播机制
+ 失效场景

## 1. 源码原理

> Spring 事务是在数据库事务的基础上进行了封装扩展，主要特性如下：
>
> + 支持原有数据库事务的隔离级别，新加入事务传播机制
> + 提供多个事务的合并或隔离功能
> + 提供声明式事务，让业务代码与事务分离（AOP）

### 1.1 TransactionDefinition 事务定义

事务的隔离级别和事务的传播行为。

```java
package org.springframework.transaction;

import org.springframework.lang.Nullable;

/**
 * Interface that defines Spring-compliant transaction properties.
 * Based on the propagation behavior definitions analogous to EJB CMT attributes.
 *
 * <p>Note that isolation level and timeout settings will not get applied unless
 * an actual new transaction gets started. As only {@link #PROPAGATION_REQUIRED},
 * {@link #PROPAGATION_REQUIRES_NEW}, and {@link #PROPAGATION_NESTED} can cause
 * that, it usually doesn't make sense to specify those settings in other cases.
 * Furthermore, be aware that not all transaction managers will support those
 * advanced features and thus might throw corresponding exceptions when given
 * non-default values.
 *
 * <p>The {@linkplain #isReadOnly() read-only flag} applies to any transaction context,
 * whether backed by an actual resource transaction or operating non-transactionally
 * at the resource level. In the latter case, the flag will only apply to managed
 * resources within the application, such as a Hibernate {@code Session}.
 *
 * @author Juergen Hoeller
 * @since 08.05.2003
 * @see PlatformTransactionManager#getTransaction(TransactionDefinition)
 * @see org.springframework.transaction.support.DefaultTransactionDefinition
 * @see org.springframework.transaction.interceptor.TransactionAttribute
 */
public interface TransactionDefinition {

    /**
     * Support a current transaction; create a new one if none exists.
     * Analogous to the EJB transaction attribute of the same name.
     * <p>This is typically the default setting of a transaction definition
     * and typically defines a transaction synchronization scope.
     */
    int PROPAGATION_REQUIRED = 0;

    /**
     * Support a current transaction; execute non-transactionally if none exists.
     * Analogous to the EJB transaction attribute of the same name.
     * <p><b>NOTE:</b> For transaction managers with transaction synchronization,
     * {@code PROPAGATION_SUPPORTS} is slightly different from no transaction
     * at all, as it defines a transaction scope that synchronization might apply to.
     * As a consequence, the same resources (a JDBC {@code Connection}, a
     * Hibernate {@code Session}, etc) will be shared for the entire specified
     * scope. Note that the exact behavior depends on the actual synchronization
     * configuration of the transaction manager.
     * <p>In general, use {@code PROPAGATION_SUPPORTS} with care. In particular, do
     * not rely on {@code PROPAGATION_REQUIRED} or {@code PROPAGATION_REQUIRES_NEW}
     * <i>within</i> a {@code PROPAGATION_SUPPORTS} scope (which may lead to
     * synchronization conflicts at runtime). If such nesting is unavoidable, make sure
     * to configure your transaction manager appropriately (typically switching to
     * "synchronization on actual transaction").
     * @see org.springframework.transaction.support.AbstractPlatformTransactionManager#setTransactionSynchronization
     * @see org.springframework.transaction.support.AbstractPlatformTransactionManager#SYNCHRONIZATION_ON_ACTUAL_TRANSACTION
     */
    int PROPAGATION_SUPPORTS = 1;

    /**
     * Support a current transaction; throw an exception if no current transaction
     * exists. Analogous to the EJB transaction attribute of the same name.
     * <p>Note that transaction synchronization within a {@code PROPAGATION_MANDATORY}
     * scope will always be driven by the surrounding transaction.
     */
    int PROPAGATION_MANDATORY = 2;

    /**
     * Create a new transaction, suspending the current transaction if one exists.
     * Analogous to the EJB transaction attribute of the same name.
     * <p><b>NOTE:</b> Actual transaction suspension will not work out-of-the-box
     * on all transaction managers. This in particular applies to
     * {@link org.springframework.transaction.jta.JtaTransactionManager},
     * which requires the {@code jakarta.transaction.TransactionManager} to be
     * made available to it (which is server-specific in standard Jakarta EE).
     * <p>A {@code PROPAGATION_REQUIRES_NEW} scope always defines its own
     * transaction synchronizations. Existing synchronizations will be suspended
     * and resumed appropriately.
     * @see org.springframework.transaction.jta.JtaTransactionManager#setTransactionManager
     */
    int PROPAGATION_REQUIRES_NEW = 3;

    /**
     * Do not support a current transaction; rather always execute non-transactionally.
     * Analogous to the EJB transaction attribute of the same name.
     * <p><b>NOTE:</b> Actual transaction suspension will not work out-of-the-box
     * on all transaction managers. This in particular applies to
     * {@link org.springframework.transaction.jta.JtaTransactionManager},
     * which requires the {@code jakarta.transaction.TransactionManager} to be
     * made available to it (which is server-specific in standard Jakarta EE).
     * <p>Note that transaction synchronization is <i>not</i> available within a
     * {@code PROPAGATION_NOT_SUPPORTED} scope. Existing synchronizations
     * will be suspended and resumed appropriately.
     * @see org.springframework.transaction.jta.JtaTransactionManager#setTransactionManager
     */
    int PROPAGATION_NOT_SUPPORTED = 4;

    /**
     * Do not support a current transaction; throw an exception if a current transaction
     * exists. Analogous to the EJB transaction attribute of the same name.
     * <p>Note that transaction synchronization is <i>not</i> available within a
     * {@code PROPAGATION_NEVER} scope.
     */
    int PROPAGATION_NEVER = 5;

    /**
     * Execute within a nested transaction if a current transaction exists,
     * behaving like {@link #PROPAGATION_REQUIRED} otherwise. There is no
     * analogous feature in EJB.
     * <p><b>NOTE:</b> Actual creation of a nested transaction will only work on
     * specific transaction managers. Out of the box, this only applies to the JDBC
     * {@link org.springframework.jdbc.datasource.DataSourceTransactionManager}
     * when working on a JDBC 3.0 driver. Some JTA providers might support
     * nested transactions as well.
     * @see org.springframework.jdbc.datasource.DataSourceTransactionManager
     */
    int PROPAGATION_NESTED = 6;


    /**
     * Use the default isolation level of the underlying datastore.
     * <p>All other levels correspond to the JDBC isolation levels.
     * @see java.sql.Connection
     */
    int ISOLATION_DEFAULT = -1;

    /**
     * Indicates that dirty reads, non-repeatable reads, and phantom reads
     * can occur.
     * <p>This level allows a row changed by one transaction to be read by another
     * transaction before any changes in that row have been committed (a "dirty read").
     * If any of the changes are rolled back, the second transaction will have
     * retrieved an invalid row.
     * @see java.sql.Connection#TRANSACTION_READ_UNCOMMITTED
     */
    int ISOLATION_READ_UNCOMMITTED = 1;  // same as java.sql.Connection.TRANSACTION_READ_UNCOMMITTED;

    /**
     * Indicates that dirty reads are prevented; non-repeatable reads and
     * phantom reads can occur.
     * <p>This level only prohibits a transaction from reading a row with uncommitted
     * changes in it.
     * @see java.sql.Connection#TRANSACTION_READ_COMMITTED
     */
    int ISOLATION_READ_COMMITTED = 2;  // same as java.sql.Connection.TRANSACTION_READ_COMMITTED;

    /**
     * Indicates that dirty reads and non-repeatable reads are prevented;
     * phantom reads can occur.
     * <p>This level prohibits a transaction from reading a row with uncommitted changes
     * in it, and it also prohibits the situation where one transaction reads a row,
     * a second transaction alters the row, and the first transaction re-reads the row,
     * getting different values the second time (a "non-repeatable read").
     * @see java.sql.Connection#TRANSACTION_REPEATABLE_READ
     */
    int ISOLATION_REPEATABLE_READ = 4;  // same as java.sql.Connection.TRANSACTION_REPEATABLE_READ;

    /**
     * Indicates that dirty reads, non-repeatable reads, and phantom reads
     * are prevented.
     * <p>This level includes the prohibitions in {@link #ISOLATION_REPEATABLE_READ}
     * and further prohibits the situation where one transaction reads all rows that
     * satisfy a {@code WHERE} condition, a second transaction inserts a row
     * that satisfies that {@code WHERE} condition, and the first transaction
     * re-reads for the same condition, retrieving the additional "phantom" row
     * in the second read.
     * @see java.sql.Connection#TRANSACTION_SERIALIZABLE
     */
    int ISOLATION_SERIALIZABLE = 8;  // same as java.sql.Connection.TRANSACTION_SERIALIZABLE;


    /**
     * Use the default timeout of the underlying transaction system,
     * or none if timeouts are not supported.
     */
    int TIMEOUT_DEFAULT = -1;


    /**
     * Return the propagation behavior.
     * <p>Must return one of the {@code PROPAGATION_XXX} constants
     * defined on {@link TransactionDefinition this interface}.
     * <p>The default is {@link #PROPAGATION_REQUIRED}.
     * @return the propagation behavior
     * @see #PROPAGATION_REQUIRED
     * @see org.springframework.transaction.support.TransactionSynchronizationManager#isActualTransactionActive()
     */
    default int getPropagationBehavior() {
        return PROPAGATION_REQUIRED;
    }

    /**
     * Return the isolation level.
     * <p>Must return one of the {@code ISOLATION_XXX} constants defined on
     * {@link TransactionDefinition this interface}. Those constants are designed
     * to match the values of the same constants on {@link java.sql.Connection}.
     * <p>Exclusively designed for use with {@link #PROPAGATION_REQUIRED} or
     * {@link #PROPAGATION_REQUIRES_NEW} since it only applies to newly started
     * transactions. Consider switching the "validateExistingTransactions" flag to
     * "true" on your transaction manager if you'd like isolation level declarations
     * to get rejected when participating in an existing transaction with a different
     * isolation level.
     * <p>The default is {@link #ISOLATION_DEFAULT}. Note that a transaction manager
     * that does not support custom isolation levels will throw an exception when
     * given any other level than {@link #ISOLATION_DEFAULT}.
     * @return the isolation level
     * @see #ISOLATION_DEFAULT
     * @see org.springframework.transaction.support.AbstractPlatformTransactionManager#setValidateExistingTransaction
     */
    default int getIsolationLevel() {
        return ISOLATION_DEFAULT;
    }

    /**
     * Return the transaction timeout.
     * <p>Must return a number of seconds, or {@link #TIMEOUT_DEFAULT}.
     * <p>Exclusively designed for use with {@link #PROPAGATION_REQUIRED} or
     * {@link #PROPAGATION_REQUIRES_NEW} since it only applies to newly started
     * transactions.
     * <p>Note that a transaction manager that does not support timeouts will throw
     * an exception when given any other timeout than {@link #TIMEOUT_DEFAULT}.
     * <p>The default is {@link #TIMEOUT_DEFAULT}.
     * @return the transaction timeout
     */
    default int getTimeout() {
        return TIMEOUT_DEFAULT;
    }

    /**
     * Return whether to optimize as a read-only transaction.
     * <p>The read-only flag applies to any transaction context, whether backed
     * by an actual resource transaction ({@link #PROPAGATION_REQUIRED}/
     * {@link #PROPAGATION_REQUIRES_NEW}) or operating non-transactionally at
     * the resource level ({@link #PROPAGATION_SUPPORTS}). In the latter case,
     * the flag will only apply to managed resources within the application,
     * such as a Hibernate {@code Session}.
     * <p>This just serves as a hint for the actual transaction subsystem;
     * it will <i>not necessarily</i> cause failure of write access attempts.
     * A transaction manager which cannot interpret the read-only hint will
     * <i>not</i> throw an exception when asked for a read-only transaction.
     * @return {@code true} if the transaction is to be optimized as read-only
     * ({@code false} by default)
     * @see org.springframework.transaction.support.TransactionSynchronization#beforeCommit(boolean)
     * @see org.springframework.transaction.support.TransactionSynchronizationManager#isCurrentTransactionReadOnly()
     */
    default boolean isReadOnly() {
        return false;
    }

    /**
     * Return the name of this transaction. Can be {@code null}.
     * <p>This will be used as the transaction name to be shown in a
     * transaction monitor, if applicable (for example, WebLogic's).
     * <p>In case of Spring's declarative transactions, the exposed name will be
     * the {@code fully-qualified class name + "." + method name} (by default).
     * @return the name of this transaction ({@code null} by default}
     * @see org.springframework.transaction.interceptor.TransactionAspectSupport
     * @see org.springframework.transaction.support.TransactionSynchronizationManager#getCurrentTransactionName()
     */
    @Nullable
    default String getName() {
        return null;
    }


    // Static builder methods

    /**
     * Return an unmodifiable {@code TransactionDefinition} with defaults.
     * <p>For customization purposes, use the modifiable
     * {@link org.springframework.transaction.support.DefaultTransactionDefinition}
     * instead.
     * @since 5.2
     */
    static TransactionDefinition withDefaults() {
        return StaticTransactionDefinition.INSTANCE;
    }

}

```

### 1.2 TransactionAttribute 事务属性

实现对回滚规则的扩展。

```java
/*
 * Copyright 2002-2019 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      https://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.springframework.transaction.interceptor;

import java.util.Collection;

import org.springframework.lang.Nullable;
import org.springframework.transaction.TransactionDefinition;

/**
 * This interface adds a {@code rollbackOn} specification to {@link TransactionDefinition}.
 * As custom {@code rollbackOn} is only possible with AOP, it resides in the AOP-related
 * transaction subpackage.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @author Mark Paluch
 * @since 16.03.2003
 * @see DefaultTransactionAttribute
 * @see RuleBasedTransactionAttribute
 */
public interface TransactionAttribute extends TransactionDefinition {

    /**
     * Return a qualifier value associated with this transaction attribute.
     * <p>This may be used for choosing a corresponding transaction manager
     * to process this specific transaction.
     * @since 3.0
     */
    @Nullable
    String getQualifier();

    /**
     * Return labels associated with this transaction attribute.
     * <p>This may be used for applying specific transactional behavior
     * or follow a purely descriptive nature.
     * @since 5.3
     */
    Collection<String> getLabels();

    /**
     * Should we roll back on the given exception?
     * @param ex the exception to evaluate
     * @return whether to perform a rollback or not
     */
    boolean rollbackOn(Throwable ex);

}

```

### 1.3 PlatformTransactionManager 平台事务管理器

```java
package org.springframework.transaction;

import org.springframework.lang.Nullable;

/**
 * This is the central interface in Spring's imperative transaction infrastructure.
 * Applications can use this directly, but it is not primarily meant as an API:
 * Typically, applications will work with either TransactionTemplate or
 * declarative transaction demarcation through AOP.
 *
 * <p>For implementors, it is recommended to derive from the provided
 * {@link org.springframework.transaction.support.AbstractPlatformTransactionManager}
 * class, which pre-implements the defined propagation behavior and takes care
 * of transaction synchronization handling. Subclasses have to implement
 * template methods for specific states of the underlying transaction,
 * for example: begin, suspend, resume, commit.
 *
 * <p>The default implementations of this strategy interface are
 * {@link org.springframework.transaction.jta.JtaTransactionManager} and
 * {@link org.springframework.jdbc.datasource.DataSourceTransactionManager},
 * which can serve as an implementation guide for other transaction strategies.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @since 16.05.2003
 * @see org.springframework.transaction.support.TransactionTemplate
 * @see org.springframework.transaction.interceptor.TransactionInterceptor
 * @see org.springframework.transaction.ReactiveTransactionManager
 */
public interface PlatformTransactionManager extends TransactionManager {

    /**
     * Return a currently active transaction or create a new one, according to
     * the specified propagation behavior.
     * <p>Note that parameters like isolation level or timeout will only be applied
     * to new transactions, and thus be ignored when participating in active ones.
     * <p>Furthermore, not all transaction definition settings will be supported
     * by every transaction manager: A proper transaction manager implementation
     * should throw an exception when unsupported settings are encountered.
     * <p>An exception to the above rule is the read-only flag, which should be
     * ignored if no explicit read-only mode is supported. Essentially, the
     * read-only flag is just a hint for potential optimization.
     * @param definition the TransactionDefinition instance (can be {@code null} for defaults),
     * describing propagation behavior, isolation level, timeout etc.
     * @return transaction status object representing the new or current transaction
     * @throws TransactionException in case of lookup, creation, or system errors
     * @throws IllegalTransactionStateException if the given transaction definition
     * cannot be executed (for example, if a currently active transaction is in
     * conflict with the specified propagation behavior)
     * @see TransactionDefinition#getPropagationBehavior
     * @see TransactionDefinition#getIsolationLevel
     * @see TransactionDefinition#getTimeout
     * @see TransactionDefinition#isReadOnly
     */
    TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
            throws TransactionException;

    /**
     * Commit the given transaction, with regard to its status. If the transaction
     * has been marked rollback-only programmatically, perform a rollback.
     * <p>If the transaction wasn't a new one, omit the commit for proper
     * participation in the surrounding transaction. If a previous transaction
     * has been suspended to be able to create a new one, resume the previous
     * transaction after committing the new one.
     * <p>Note that when the commit call completes, no matter if normally or
     * throwing an exception, the transaction must be fully completed and
     * cleaned up. No rollback call should be expected in such a case.
     * <p>If this method throws an exception other than a TransactionException,
     * then some before-commit error caused the commit attempt to fail. For
     * example, an O/R Mapping tool might have tried to flush changes to the
     * database right before commit, with the resulting DataAccessException
     * causing the transaction to fail. The original exception will be
     * propagated to the caller of this commit method in such a case.
     * @param status object returned by the {@code getTransaction} method
     * @throws UnexpectedRollbackException in case of an unexpected rollback
     * that the transaction coordinator initiated
     * @throws HeuristicCompletionException in case of a transaction failure
     * caused by a heuristic decision on the side of the transaction coordinator
     * @throws TransactionSystemException in case of commit or system errors
     * (typically caused by fundamental resource failures)
     * @throws IllegalTransactionStateException if the given transaction
     * is already completed (that is, committed or rolled back)
     * @see TransactionStatus#setRollbackOnly
     */
    void commit(TransactionStatus status) throws TransactionException;

    /**
     * Perform a rollback of the given transaction.
     * <p>If the transaction wasn't a new one, just set it rollback-only for proper
     * participation in the surrounding transaction. If a previous transaction
     * has been suspended to be able to create a new one, resume the previous
     * transaction after rolling back the new one.
     * <p><b>Do not call rollback on a transaction if commit threw an exception.</b>
     * The transaction will already have been completed and cleaned up when commit
     * returns, even in case of a commit exception. Consequently, a rollback call
     * after commit failure will lead to an IllegalTransactionStateException.
     * @param status object returned by the {@code getTransaction} method
     * @throws TransactionSystemException in case of rollback or system errors
     * (typically caused by fundamental resource failures)
     * @throws IllegalTransactionStateException if the given transaction
     * is already completed (that is, committed or rolled back)
     */
    void rollback(TransactionStatus status) throws TransactionException;

}

```

![依赖关系](/assets/img/2023-10-04-spring-transaction/2023-10-04-18-19-33.png)

### 1.4 TransactionStatus 事务运行时状态

```java
package org.springframework.transaction;

import java.io.Flushable;

/**
 * Representation of the status of a transaction.
 *
 * <p>Transactional code can use this to retrieve status information,
 * and to programmatically request a rollback (instead of throwing
 * an exception that causes an implicit rollback).
 *
 * <p>Includes the {@link SavepointManager} interface to provide access
 * to savepoint management facilities. Note that savepoint management
 * is only available if supported by the underlying transaction manager.
 *
 * @author Juergen Hoeller
 * @since 27.03.2003
 * @see #setRollbackOnly()
 * @see PlatformTransactionManager#getTransaction
 * @see org.springframework.transaction.support.TransactionCallback#doInTransaction
 * @see org.springframework.transaction.interceptor.TransactionInterceptor#currentTransactionStatus()
 */
public interface TransactionStatus extends TransactionExecution, SavepointManager, Flushable {

    /**
     * Return whether this transaction internally carries a savepoint,
     * that is, has been created as nested transaction based on a savepoint.
     * <p>This method is mainly here for diagnostic purposes, alongside
     * {@link #isNewTransaction()}. For programmatic handling of custom
     * savepoints, use the operations provided by {@link SavepointManager}.
     * @see #isNewTransaction()
     * @see #createSavepoint()
     * @see #rollbackToSavepoint(Object)
     * @see #releaseSavepoint(Object)
     */
    boolean hasSavepoint();

    /**
     * Flush the underlying session to the datastore, if applicable:
     * for example, all affected Hibernate/JPA sessions.
     * <p>This is effectively just a hint and may be a no-op if the underlying
     * transaction manager does not have a flush concept. A flush signal may
     * get applied to the primary resource or to transaction synchronizations,
     * depending on the underlying resource.
     */
    @Override
    void flush();

}

```

### 1.5 TransactionInterceptor 事务拦截器

**核心方法：** `TransactionInterceptor#invoke()`

```java
package org.springframework.transaction.interceptor;

import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;
import java.util.Properties;

import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;

import org.springframework.aop.support.AopUtils;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.lang.Nullable;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.TransactionManager;

/**
 * AOP Alliance MethodInterceptor for declarative transaction
 * management using the common Spring transaction infrastructure
 * ({@link org.springframework.transaction.PlatformTransactionManager}/
 * {@link org.springframework.transaction.ReactiveTransactionManager}).
 *
 * <p>Derives from the {@link TransactionAspectSupport} class which
 * contains the integration with Spring's underlying transaction API.
 * TransactionInterceptor simply calls the relevant superclass methods
 * such as {@link #invokeWithinTransaction} in the correct order.
 *
 * <p>TransactionInterceptors are thread-safe.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @author Sebastien Deleuze
 * @see TransactionProxyFactoryBean
 * @see org.springframework.aop.framework.ProxyFactoryBean
 * @see org.springframework.aop.framework.ProxyFactory
 */
@SuppressWarnings("serial")
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {

    /**
     * Create a new TransactionInterceptor.
     * <p>Transaction manager and transaction attributes still need to be set.
     * @see #setTransactionManager
     * @see #setTransactionAttributes(java.util.Properties)
     * @see #setTransactionAttributeSource(TransactionAttributeSource)
     */
    public TransactionInterceptor() {
    }

    /**
     * Create a new TransactionInterceptor.
     * @param ptm the default transaction manager to perform the actual transaction management
     * @param tas the attribute source to be used to find transaction attributes
     * @since 5.2.5
     * @see #setTransactionManager
     * @see #setTransactionAttributeSource
     */
    public TransactionInterceptor(TransactionManager ptm, TransactionAttributeSource tas) {
        setTransactionManager(ptm);
        setTransactionAttributeSource(tas);
    }

    /**
     * Create a new TransactionInterceptor.
     * @param ptm the default transaction manager to perform the actual transaction management
     * @param tas the attribute source to be used to find transaction attributes
     * @see #setTransactionManager
     * @see #setTransactionAttributeSource
     * @deprecated as of 5.2.5, in favor of
     * {@link #TransactionInterceptor(TransactionManager, TransactionAttributeSource)}
     */
    @Deprecated
    public TransactionInterceptor(PlatformTransactionManager ptm, TransactionAttributeSource tas) {
        setTransactionManager(ptm);
        setTransactionAttributeSource(tas);
    }

    /**
     * Create a new TransactionInterceptor.
     * @param ptm the default transaction manager to perform the actual transaction management
     * @param attributes the transaction attributes in properties format
     * @see #setTransactionManager
     * @see #setTransactionAttributes(java.util.Properties)
     * @deprecated as of 5.2.5, in favor of {@link #setTransactionAttributes(Properties)}
     */
    @Deprecated
    public TransactionInterceptor(PlatformTransactionManager ptm, Properties attributes) {
        setTransactionManager(ptm);
        setTransactionAttributes(attributes);
    }


    @Override
    @Nullable
    public Object invoke(MethodInvocation invocation) throws Throwable {
        // Work out the target class: may be {@code null}.
        // The TransactionAttributeSource should be passed the target class
        // as well as the method, which may be from an interface.
        Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

        // Adapt to TransactionAspectSupport's invokeWithinTransaction...
        return invokeWithinTransaction(invocation.getMethod(), targetClass, new CoroutinesInvocationCallback() {
            @Override
            @Nullable
            public Object proceedWithInvocation() throws Throwable {
                return invocation.proceed();
            }
            @Override
            public Object getTarget() {
                return invocation.getThis();
            }
            @Override
            public Object[] getArguments() {
                return invocation.getArguments();
            }
        });
    }


    //---------------------------------------------------------------------
    // Serialization support
    //---------------------------------------------------------------------

    private void writeObject(ObjectOutputStream oos) throws IOException {
        // Rely on default serialization, although this class itself doesn't carry state anyway...
        oos.defaultWriteObject();

        // Deserialize superclass fields.
        oos.writeObject(getTransactionManagerBeanName());
        oos.writeObject(getTransactionManager());
        oos.writeObject(getTransactionAttributeSource());
        oos.writeObject(getBeanFactory());
    }

    private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        // Rely on default serialization, although this class itself doesn't carry state anyway...
        ois.defaultReadObject();

        // Serialize all relevant superclass fields.
        // Superclass can't implement Serializable because it also serves as base class
        // for AspectJ aspects (which are not allowed to implement Serializable)!
        setTransactionManagerBeanName((String) ois.readObject());
        setTransactionManager((PlatformTransactionManager) ois.readObject());
        setTransactionAttributeSource((TransactionAttributeSource) ois.readObject());
        setBeanFactory((BeanFactory) ois.readObject());
    }

}
```

### 1.6 编程式事务和声明式事务

**编程式事务**

```java
package priv.utrix.exam.utils;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.support.TransactionTemplate;

import java.util.function.Supplier;

/**
 * 事务工具类
 *
 * @author Ori Ming
 */
@Slf4j
@Configuration
@RequiredArgsConstructor
public class TransactionUtil {

   private final TransactionTemplate template;

    /**
     * 手动执行事务, 简化事务覆盖面
     *
     * @param task 执行任务，并自定义返回值
     * @return U 自定义返回对象
     */
    public <U> U execute(Supplier<U> task) {
        return template.execute(transactionStatus -> {
            try {
                return task.get();
            } catch (Exception e) {
                // 自动捕捉异常，回滚
                log.error("任务异常, 回归事务: ", e);
                transactionStatus.setRollbackOnly();
                throw e;
            }
        });
    }
}

```

**声明式事务**
使用注解 `@Transactional` 的方式

```xml
<!-- Spring 开启事务控制的注解支持 -->
<tx:annotation-driven transaction-manager="txManager"/>
```

注解属性:

```java
package org.springframework.transaction.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import org.springframework.core.annotation.AliasFor;
import org.springframework.transaction.TransactionDefinition;

/**
 * Describes a transaction attribute on an individual method or on a class.
 *
 * <p>When this annotation is declared at the class level, it applies as a default
 * to all methods of the declaring class and its subclasses. Note that it does not
 * apply to ancestor classes up the class hierarchy; inherited methods need to be
 * locally redeclared in order to participate in a subclass-level annotation. For
 * details on method visibility constraints, consult the
 * <a href="https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction">Transaction Management</a>
 * section of the reference manual.
 *
 * <p>This annotation is generally directly comparable to Spring's
 * {@link org.springframework.transaction.interceptor.RuleBasedTransactionAttribute}
 * class, and in fact {@link AnnotationTransactionAttributeSource} will directly
 * convert this annotation's attributes to properties in {@code RuleBasedTransactionAttribute},
 * so that Spring's transaction support code does not have to know about annotations.
 *
 * <h3>Attribute Semantics</h3>
 *
 * <p>If no custom rollback rules are configured in this annotation, the transaction
 * will roll back on {@link RuntimeException} and {@link Error} but not on checked
 * exceptions.
 *
 * <p>Rollback rules determine if a transaction should be rolled back when a given
 * exception is thrown, and the rules are based on patterns. A pattern can be a
 * fully qualified class name or a substring of a fully qualified class name for
 * an exception type (which must be a subclass of {@code Throwable}), with no
 * wildcard support at present. For example, a value of
 * {@code "javax.servlet.ServletException"} or {@code "ServletException"} will
 * match {@code javax.servlet.ServletException} and its subclasses.
 *
 * <p>Rollback rules may be configured via {@link #rollbackFor}/{@link #noRollbackFor}
 * and {@link #rollbackForClassName}/{@link #noRollbackForClassName}, which allow
 * patterns to be specified as {@link Class} references or {@linkplain String
 * strings}, respectively. When an exception type is specified as a class reference
 * its fully qualified name will be used as the pattern. Consequently,
 * {@code @Transactional(rollbackFor = example.CustomException.class)} is equivalent
 * to {@code @Transactional(rollbackForClassName = "example.CustomException")}.
 *
 * <p><strong>WARNING:</strong> You must carefully consider how specific the pattern
 * is and whether to include package information (which isn't mandatory). For example,
 * {@code "Exception"} will match nearly anything and will probably hide other
 * rules. {@code "java.lang.Exception"} would be correct if {@code "Exception"}
 * were meant to define a rule for all checked exceptions. With more unique
 * exception names such as {@code "BaseBusinessException"} there is likely no
 * need to use the fully qualified class name for the exception pattern. Furthermore,
 * rollback rules may result in unintentional matches for similarly named exceptions
 * and nested classes. This is due to the fact that a thrown exception is considered
 * to be a match for a given rollback rule if the name of thrown exception contains
 * the exception pattern configured for the rollback rule. For example, given a
 * rule configured to match on {@code com.example.CustomException}, that rule
 * would match against an exception named
 * {@code com.example.CustomExceptionV2} (an exception in the same package as
 * {@code CustomException} but with an additional suffix) or an exception named
 * {@code com.example.CustomException$AnotherException}
 * (an exception declared as a nested class in {@code CustomException}).
 *
 * <p>For specific information about the semantics of other attributes in this
 * annotation, consult the {@link org.springframework.transaction.TransactionDefinition}
 * and {@link org.springframework.transaction.interceptor.TransactionAttribute} javadocs.
 *
 * <h3>Transaction Management</h3>
 *
 * <p>This annotation commonly works with thread-bound transactions managed by a
 * {@link org.springframework.transaction.PlatformTransactionManager}, exposing a
 * transaction to all data access operations within the current execution thread.
 * <b>Note: This does NOT propagate to newly started threads within the method.</b>
 *
 * <p>Alternatively, this annotation may demarcate a reactive transaction managed
 * by a {@link org.springframework.transaction.ReactiveTransactionManager} which
 * uses the Reactor context instead of thread-local variables. As a consequence,
 * all participating data access operations need to execute within the same
 * Reactor context in the same reactive pipeline.
 *
 * @author Colin Sampaleanu
 * @author Juergen Hoeller
 * @author Sam Brannen
 * @author Mark Paluch
 * @since 1.2
 * @see org.springframework.transaction.interceptor.TransactionAttribute
 * @see org.springframework.transaction.interceptor.DefaultTransactionAttribute
 * @see org.springframework.transaction.interceptor.RuleBasedTransactionAttribute
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {

    /**
     * Alias for {@link #transactionManager}.
     * @see #transactionManager
     */
    @AliasFor("transactionManager")
    String value() default "";

    /**
     * A <em>qualifier</em> value for the specified transaction.
     * <p>May be used to determine the target transaction manager, matching the
     * qualifier value (or the bean name) of a specific
     * {@link org.springframework.transaction.TransactionManager TransactionManager}
     * bean definition.
     * @since 4.2
     * @see #value
     * @see org.springframework.transaction.PlatformTransactionManager
     * @see org.springframework.transaction.ReactiveTransactionManager
     */
    @AliasFor("value")
    String transactionManager() default "";

    /**
     * Defines zero (0) or more transaction labels.
     * <p>Labels may be used to describe a transaction, and they can be evaluated
     * by individual transaction managers. Labels may serve a solely descriptive
     * purpose or map to pre-defined transaction manager-specific options.
     * <p>See the documentation of the actual transaction manager implementation
     * for details on how it evaluates transaction labels.
     * @since 5.3
     * @see org.springframework.transaction.interceptor.DefaultTransactionAttribute#getLabels()
     */
    String[] label() default {};

    /**
     * The transaction propagation type.
     * <p>Defaults to {@link Propagation#REQUIRED}.
     * @see org.springframework.transaction.interceptor.TransactionAttribute#getPropagationBehavior()
     */
    Propagation propagation() default Propagation.REQUIRED;

    /**
     * The transaction isolation level.
     * <p>Defaults to {@link Isolation#DEFAULT}.
     * <p>Exclusively designed for use with {@link Propagation#REQUIRED} or
     * {@link Propagation#REQUIRES_NEW} since it only applies to newly started
     * transactions. Consider switching the "validateExistingTransactions" flag to
     * "true" on your transaction manager if you'd like isolation level declarations
     * to get rejected when participating in an existing transaction with a different
     * isolation level.
     * @see org.springframework.transaction.interceptor.TransactionAttribute#getIsolationLevel()
     * @see org.springframework.transaction.support.AbstractPlatformTransactionManager#setValidateExistingTransaction
     */
    Isolation isolation() default Isolation.DEFAULT;

    /**
     * The timeout for this transaction (in seconds).
     * <p>Defaults to the default timeout of the underlying transaction system.
     * <p>Exclusively designed for use with {@link Propagation#REQUIRED} or
     * {@link Propagation#REQUIRES_NEW} since it only applies to newly started
     * transactions.
     * @return the timeout in seconds
     * @see org.springframework.transaction.interceptor.TransactionAttribute#getTimeout()
     */
    int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;

    /**
     * The timeout for this transaction (in seconds).
     * <p>Defaults to the default timeout of the underlying transaction system.
     * <p>Exclusively designed for use with {@link Propagation#REQUIRED} or
     * {@link Propagation#REQUIRES_NEW} since it only applies to newly started
     * transactions.
     * @return the timeout in seconds as a String value, e.g. a placeholder
     * @since 5.3
     * @see org.springframework.transaction.interceptor.TransactionAttribute#getTimeout()
     */
    String timeoutString() default "";

    /**
     * A boolean flag that can be set to {@code true} if the transaction is
     * effectively read-only, allowing for corresponding optimizations at runtime.
     * <p>Defaults to {@code false}.
     * <p>This just serves as a hint for the actual transaction subsystem;
     * it will <i>not necessarily</i> cause failure of write access attempts.
     * A transaction manager which cannot interpret the read-only hint will
     * <i>not</i> throw an exception when asked for a read-only transaction
     * but rather silently ignore the hint.
     * @see org.springframework.transaction.interceptor.TransactionAttribute#isReadOnly()
     * @see org.springframework.transaction.support.TransactionSynchronizationManager#isCurrentTransactionReadOnly()
     */
    boolean readOnly() default false;

    /**
     * Defines zero (0) or more exception {@linkplain Class classes}, which must be
     * subclasses of {@link Throwable}, indicating which exception types must cause
     * a transaction rollback.
     * <p>By default, a transaction will be rolled back on {@link RuntimeException}
     * and {@link Error} but not on checked exceptions (business exceptions). See
     * {@link org.springframework.transaction.interceptor.DefaultTransactionAttribute#rollbackOn(Throwable)}
     * for a detailed explanation.
     * <p>This is the preferred way to construct a rollback rule (in contrast to
     * {@link #rollbackForClassName}), matching the exception type, its subclasses,
     * and its nested classes. See the {@linkplain Transactional class-level javadocs}
     * for further details on rollback rule semantics and warnings regarding possible
     * unintentional matches.
     * @see #rollbackForClassName
     * @see org.springframework.transaction.interceptor.RollbackRuleAttribute#RollbackRuleAttribute(Class)
     * @see org.springframework.transaction.interceptor.DefaultTransactionAttribute#rollbackOn(Throwable)
     */
    Class<? extends Throwable>[] rollbackFor() default {};

    /**
     * Defines zero (0) or more exception name patterns (for exceptions which must be a
     * subclass of {@link Throwable}), indicating which exception types must cause
     * a transaction rollback.
     * <p>See the {@linkplain Transactional class-level javadocs} for further details
     * on rollback rule semantics, patterns, and warnings regarding possible
     * unintentional matches.
     * @see #rollbackFor
     * @see org.springframework.transaction.interceptor.RollbackRuleAttribute#RollbackRuleAttribute(String)
     * @see org.springframework.transaction.interceptor.DefaultTransactionAttribute#rollbackOn(Throwable)
     */
    String[] rollbackForClassName() default {};

    /**
     * Defines zero (0) or more exception {@link Class Classes}, which must be
     * subclasses of {@link Throwable}, indicating which exception types must
     * <b>not</b> cause a transaction rollback.
     * <p>This is the preferred way to construct a rollback rule (in contrast to
     * {@link #noRollbackForClassName}), matching the exception type, its subclasses,
     * and its nested classes. See the {@linkplain Transactional class-level javadocs}
     * for further details on rollback rule semantics and warnings regarding possible
     * unintentional matches.
     * @see #noRollbackForClassName
     * @see org.springframework.transaction.interceptor.NoRollbackRuleAttribute#NoRollbackRuleAttribute(Class)
     * @see org.springframework.transaction.interceptor.DefaultTransactionAttribute#rollbackOn(Throwable)
     */
    Class<? extends Throwable>[] noRollbackFor() default {};

    /**
     * Defines zero (0) or more exception name patterns (for exceptions which must be a
     * subclass of {@link Throwable}) indicating which exception types must <b>not</b>
     * cause a transaction rollback.
     * <p>See the {@linkplain Transactional class-level javadocs} for further details
     * on rollback rule semantics, patterns, and warnings regarding possible
     * unintentional matches.
     * @see #noRollbackFor
     * @see org.springframework.transaction.interceptor.NoRollbackRuleAttribute#NoRollbackRuleAttribute(String)
     * @see org.springframework.transaction.interceptor.DefaultTransactionAttribute#rollbackOn(Throwable)
     */
    String[] noRollbackForClassName() default {};

}

```

## 2.事务传播机制

Spring 在 TransactionDefinition 接口中定义了七个事务传播行为：

+ propagation_requierd：如果当前没有事务，就新建一个事务，如果已存在一个事务中，加入到这个事务中，这是最常见的选择。
+ propagation_supports：支持当前事务，如果没有当前事务，就以非事务方法执行。
+ propagation_mandatory：使用当前事务，如果没有当前事务，就抛出异常。
+ propagation_required_new：新建事务，如果当前存在事务，把当前事务挂起。
+ propagation_not_supported：以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。
+ propagation_never：以非事务方式执行操作，如果当前事务存在则抛出异常。
+ propagation_nested：如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与propagation_required类似的操作

**常用事务传播机制：**

1. **PROPAGATION_REQUIRED**

    这个也是默认的传播机制；

2. **PROPAGATION_REQUIRES_NEW**

    总是新启一个事务，这个传播机制适用于不受父方法事务影响的操作，比如某些业务场景下需要记录业务日志，用于异步反查，那么不管主体业务逻辑是否完成，日志都需要记录下来，不能因为主体业务逻辑报错而丢失日志；

3. **PROPAGATION_NOT_SUPPORTED**

    可以用于发送提示消息，站内信、短信、邮件提示等。不属于并且不应当影响主体业务逻辑，即使发送失败也不应该对主体业务逻辑回滚。

## 3. 事务失效场景

1. Bean 是否是代理对象，即是否注入到了 IOC 容器中；
2. 函数访问级别是 Public，final 修饰，因为基于 AOP 实现的；
3. 数据库引擎是否支持事务（MySQL中 MyIsam 不支持事务），行锁才支持事务；
4. 切点是否配置正确;
5. 内部方法调用，导致事务失效。
    + 使用注入自身（`@Autowired、@Resource`）的方式调用;
    + 通过 `AopContext.currentProxy()` 获取当前代理对象调用的方式；
6. 异常类型配置是否正确;
7. 设置错误的传播机制；
8. 抛出的异常被自己 catch 了；
9. `ApplicationListener` 异步导致主业务事务没有覆盖到 Listener 的逻辑；

<br/>
<br/>

核心内容摘自：

+ wego_xuchang <https://juejin.cn/post/6989900968833843231>
