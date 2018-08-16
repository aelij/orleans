﻿---
layout: page
title: Transactions in Orleans 2.0
---

# Orleans Transactions (Beta)

__Orleans transactions are in a beta state and should not be used in production systems.__

Orleans supports distributed transactions against persistent grain state.

## Setup

Orleans transactions are opt-in.  A silo must be configured to use transactions.  If it is not, any calls to transactional grains will receive an OrleansTransactionsDisabledException.

### Transaction Manager

Orleans transactions requires a transaction manager, which is the authoritative source of information about the state of a transaction.  Currently, the transaction manager is run within the cluster and must be configured.  This can be programmatically configured on the silo host builder using UseInClusterTransactionManager and TransactionsConfiguration.

The TransactionsConfiguration contains the following configurable values:
* TransactionIdAllocationBatchSize - To avoid a storage call on every transaction start, transaction Ids are allocated in batches. This is the number of new transaction Ids allocated per transaction Id generation request (which does a storage call).
* AvailableTransactionIdThreshold - A new batch of transaction Ids will be automatically allocated if the number of available ids drops below this threshold.
* TransactionRecordPreservationDuration - How long to preserve a transaction record in the TM memory after the transaction has completed.  This is used to answer queries about the outcome of the transaction.

It is suggested that you use the defaults for these values unless you have evidence they are insufficient.

### Transaction Log

The transaction manager must persist transaction results for recoverability purposes.  To support a range of backend storage solutions an abstraction has been provided for log storage.  We currently support an Azure storage version of the log storage, and an in-memory version for development and test purposes.  This can be programmatically configured on the silo host builder using UseAzureTransactionLog or UseInMemoryTransactionLog.

### Transactional State

To support various transactional storage patterns, the grain-facing interface and supporting logic is pluggable.  We currently only support ITransactionalState.  This can be programmatically enabled on the silo host builder using UseTransactionalState.  To use a transactional state, a storage provider also needs to be configured.

Example:

``` csharp
var builder = new SiloHostBuilder()
    [...]
    .AddAzureTableGrainStorageAsDefault(options => options.ConnectionString = ”YOUR_STORAGE_CONNECTION_STRING”)
    .UseInClusterTransactionManager()
    .UseAzureTransactionLog(options => options.ConnectionString = ”YOUR_STORAGE_CONNECTION_STRING”) 
    .UseTransactionalState();

var host = builder.Build();
await host.StartAsync();
```

## Programming Model

For a grain to support transactions, transactional operations on a grain must be marked as being part of a transaction using the “Transaction” attribute.  A call can be marked as “Required”, meaning that the call will take part in the caller's transaction, or if the caller is not running a transaction, then the call will start a new transaction. If a call is marked as “RequiresNew”, then it will always start a new transaction, even if the caller is executing within a transaction.

Example:

``` csharp
public interface IATMGrain : IGrainWithIntegerKey
{
    [Transaction(TransactionOption.RequiresNew)]
    Task Transfer(Guid fromAccount, Guid toAccount, uint amountToTransfer);
}
```

The Transfer operation in the above ATM grain will always start a new transaction that involves the two referenced accounts.

``` csharp
public interface IAccountGrain : IGrainWithGuidKey
{
    [Transaction(TransactionOption.Required)]
    Task Withdraw(uint amount);

    [Transaction(TransactionOption.Required)]
    Task Deposit(uint amount);

    [Transaction(TransactionOption.Required)]
    Task<uint> GetBalance();
}
```

The Withdraw and Deposit operations in the above account grain can take part in other transactional grains' transactional operations, such as the Transfer operation in the ATM grain.

To use a transactional state within a grain, one need only define a serializable state class to be persisted and declare the state in the grain's constructor.

Example:

``` csharp
public class AccountGrain : Grain, IAccountGrain
{
    private readonly ITransactionalState<Balance> balance;

    public AccountGrain(
        [TransactionalState("balance")] ITransactionalState<Balance> balance)
    {
        this.balance = balance ?? throw new ArgumentNullException(nameof(balance));
    }

    Task IAccountGrain.Deposit(uint ammount)
    {
        this.balance.State.Value += ammount;
        this.balance.Save();
        return Task.CompletedTask;
    }

    Task IAccountGrain.Withdrawal(uint ammount)
    {
        this.balance.State.Value -= ammount;
        this.balance.Save();
        return Task.CompletedTask;
    }

    Task<uint> IAccountGrain.GetBalance()
    {
        return Task.FromResult(this.balance.State.Value);
    }
}
```

In the above example the attribute TransactionalState is used to declare that the ‘balance’ construction argument should be associated with a transactional state named “balance”.  With this declaration Orleans will wire up an ITransactionalState<Balance> instance with a state loaded from the default storage provider for the grain to use.  The state can be accessed and modified via the State property.  Changes to the state need be recorded by calling ‘Save()’.  The transaction infrastructure will ensure that any such changes performed as part of a transaction, even among multiple grains distributed over an Orleans cluster, will either all be committed or all be undone.
