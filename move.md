# Move White Paper

A safe and flexible programming language for the Libra Blockchain

1. Introduction

* Challenges of representing digital asserts on a blockchain
* How to design Move
* Move's key features and programming model
* Technical details and virtual machine design
* The progress and plan

2. Managing Digital Asserts on a Blockchain

Explaining the blockchain at an abstract level.

2.1. Abstruct view about blockchain

A blockchain is a relicated state machine.

Each validator understands how to execute a transaction to transition its internal state machine from the current state to a new state.

2.2. Encoding Digital Asserts in an Opening System

Deside how transitions and state are represented.

* Scarcity
* Access control

2.3. Existing Blockchain Language

* Indirect representation of asserts.
* Scarity is not extensible
* Access control is not flexible

3. Move Design Goals

first-class assets, flexibility, safety, and verifiability

3.1 First-Class Resources

Define custom resource types with semantics inspired by linear logic.
A resource can never be copied or implicity discarded, only moved between program storage locations.

A module declares resource types and procedures that encode the rules for creating, destroying, and updating its declared resources.
Modules enforce strong data abstraction.

3.2. Flexibility

Transaction script is effectively the main procedure of the transaction.

modules/resources/procedures

3.3. Safety

The bytecode is checked on-chain for resource, type, and memory safety by a bytecode verifier and then executed directly by a bytecode interpreter

3.4. Verifiability

More amenable to static verification

* No dynamic dispatch.
* Limited mutability.
* Modularity.

4. Move Overview

4.1. Peer-to-Peer Payment Transaction Script

```
public main(payee: address, amount: u64) {
  let coin: 0x0.Currency.Coin = 0x0.Currency.withdraw_from_sender(copy(amount));
  0x0.Currency.deposit(copy(payee), move(coin));
}
```

Amount coins will be transferred from the transaction sender to payee.

Resources must be moved exactly once.

4.2 Currency Module

* Primer: Move execution model.
    - transaction scripts
    - modules

* Declaring the Coin resource.

```
  module Currency {
  resource Coin { value: u64 }
  // ...
}
```

Module authors have complete control over the access, creation, and destruction of their declared resources.

* Implementing deposit.

```
public deposit(payee: address, to_deposit: Coin) {
  let to_deposit_value: u64 = Unpack<Coin>(move(to_deposit));
  let coin_ref: &mut Coin = BorrowGlobal<Coin>(move(payee));
  let coin_value_ref: &mut u64 = &mut move(coin_ref).value;
  let coin_value: u64 = *move(coin_value_ref);
  *move(coin_value_ref) = move(coin_value) + move(to_deposit_value);
}
```

1. Destorying the input Coin and recording its value.
2. Acquring a reference to the unique Coin resource stored under the payee's account.
3. Incrementing the value of payee's Coin by the value of the Coin passed to the procedure.

Unpanck<T> is the only way to delete a resource of type T.
BorrowGlobal<T> takes an address as input and returns a reference to the unique instance of T published under that address.

* Implementing withdraw_from_sender.

```
public withdraw_from_sender(amount: u64): Coin {
  let transaction_sender_address: address = GetTxnSenderAddress();
  let coin_ref: &mut Coin = BorrowGlobal<Coin>(move(transaction_sender_address));
  let coin_value_ref: &mut u64 = &mut move(coin_ref).value;
  let coin_value: u64 = *move(coin_value_ref);
  RejectUnless(copy(coin_value) >= copy(amount));
  *move(coin_value_ref) = move(coin_value) - copy(amount);
  let new_coin: Coin = Pack<Coin>(move(amount));
  return move(new_coin);
}
```

1. Acquires a reference to the uniqure resource of type Coin published under the sender's account.
2. Decreases the value of the referenced Coin by the input amount.
3. Creates and returns a new Coin with value amount.

5. The Move Language

* Global state

```
Σ ∈ GlobalState = AccountAddress ⇀ Account
Account = (StructID ⇀ Resource) × (ModuleName ⇀ Module)
```

* Modules

```
Module = ModuleName × (StructName ⇀ StructDecl)
                    × (ProcedureName ⇀ ProcedureDecl)
ModuleID = AccountAddress × ModuleName
StructID = ModuleID × StructName
StructDecl = Kind × (FieldName ⇀ NonReferenceType)
```

* Types

```
PrimitiveType = AccountAddress ∪ Bool ∪ UnsignedInt64 ∪ Bytes
StructType = StructID × Kind
T ⊆ NonReferenceType = StructType ∪ PrimitiveType
Type ::= T | &mut T | & T
```

* Values

Supports reference values.
A reference must be created during the execution of a transaction script and released before the end of that transaction script.

* Procedures and transaction scripts.

A procedure signature consists of visibility, typed formal parameters, and return types.
Procedure visibility may be either public or internal.
A transaction script is simply a procedure declaration with no associated module.
The dependency relationship among modules is acyclic by construction.

5.1. Bytecode Interpreter
