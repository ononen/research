# storage and state of cita-vm(cita-trie) and cita-executor & cita-chain

## the struct

*Use the cache of cita-vm, so we don't care the cache part for now.*

### [StateDB] from cita-executor:

```rust
pub struct StateDB {
    /// Backing database.
    db: Box<JournalDB>,
    /// Shared canonical state cache.
    account_cache: Arc<Mutex<AccountCache>>,
    /// DB Code cache. Maps code hashes to shared bytes.
    code_cache: Arc<Mutex<MemoryLruCache<H256, Arc<Vec<u8>>>>>,
    /// Local dirty account cache.
    local_account_cache: Vec<CacheQueueItem>,
    /// Shared account bloom. Does not handle chain reorganizations.
    account_bloom: Arc<Mutex<Bloom>>,
    cache_size: usize,
    /// Hash of the block on top of which this instance was created or
    /// `None` if cache is disabled
    pub parent_hash: Option<H256>,
    /// Hash of the committing block or `None` if not committed yet.
    commit_hash: Option<H256>,
    /// Number of the committing block or `None` if not committed yet.
    commit_number: Option<BlockNumber>,
}
```

except cache, what we need is:

```
db: Box<JournalDB>,
// Journaling bloom filter form Parity
account_bloom: Arc<Mutex<Bloom>>,
commit_hash: Option<H256>,
commit_number: Option<BlockNumber>,
```

* db: JournalDB
* account_bloom: Journaling bloom filter form Parity. it is used to handle request for the non-existant account fast
* commit_hash
* commit_number

### [executor-State] from cita-executor:

```rust
pub struct State<B: Backend> {
    db: B,
    root: H256,
    cache: RefCell<HashMap<Address, AccountEntry>>,
    // The original account is preserved in
    checkpoints: RefCell<Vec<HashMap<Address, Option<AccountEntry>>>>,
    account_start_nonce: U256,
    factories: Factories,
    pub super_admin_account: Option<Address>,
}
```

* account_start_nonce:

It's the problem when `new_contract` :

```
Account::new_contract(balance, self.account_start_nonce + nonce_offset)
```

while cita-vm:

```
let mut state_object = StateObject::new(balance, nonce);
```

* factories:

```
/// Collection of factories.
#[derive(Default, Clone)]
pub struct Factories {
    /// factory for evm.
    pub vm: EvmFactory,
    pub native: NativeFactory,
    /// factory for tries.
    pub trie: TrieFactory,
    /// factory for account databases.
    pub accountdb: AccountFactory,
}
```

* super_admin_account:

It's only used for `amend` operation

```
/// Representation of the entire state of all accounts in the system.
///
/// `State` can work together with `StateDB` to share account cache.
///
/// Local cache contains changes made locally and changes accumulated
/// locally from previous commits. Global cache reflects the database
/// state and never contains any changes.
///
/// State checkpointing.
///
/// A new checkpoint can be created with `checkpoint()`. checkpoints can be
/// created in a hierarchy.
/// When a checkpoint is active all changes are applied directly into
/// `cache` and the original value is copied into an active checkpoint.
```

### [State] from cita-vm

```rust
/// State is the one who managers all accounts and states in Ethereum's system.
pub struct State<B> {
    pub db: Arc<B>,
    pub root: H256,
    pub cache: RefCell<HashMap<Address, StateObjectEntry>>,
    /// Checkpoints are used to revert to history.
    pub checkpoints: RefCell<Vec<HashMap<Address, Option<StateObjectEntry>>>>,
}
```

except cache, what we need is:

```
db: Arc<B>,
root: H256
checkpoints: RefCell<Vec<HashMap<Address, Option<StateObjectEntry>>>>,
```

* db:
* root: for creating new state with existing state root.
* checkpoints: used to revert to history.

### [cita-Column] from cita-chain

```rust
// database columns
/// Column for State
pub const COL_STATE: Option<u32> = Some(0);
/// Column for Block headers
pub const COL_HEADERS: Option<u32> = Some(1);
/// Column for Block bodies
pub const COL_BODIES: Option<u32> = Some(2);
/// Column for Extras
pub const COL_EXTRA: Option<u32> = Some(3);
/// Column for Traces
pub const COL_TRACE: Option<u32> = Some(4);
/// Column for the empty accounts bloom filter.
pub const COL_ACCOUNT_BLOOM: Option<u32> = Some(5);
/// Column for general information from the local node which can persist.
pub const COL_NODE_INFO: Option<u32> = Some(6);
```

* COL_EXTRA: current block hash, transaction address, block receipt, block bloom, block head hash, block body hash.
* COL_TRACE: trace db?
* COL_ACCOUNT_BLOOM: account bloom, only used for snapshot
* COL_NODE_INFO: useless, remove it

**need to rewrite the rocksdb part.**

### The interface

database

* transaction
* get
* get_by_prefix
* write_buffered
* write
* flush
* iter
* iter_from_prefix
* restore
* close

**consider to redefine the iterface**

rocksdb:

* ~~open_default~~
* open

**used for cita-chain and cita-executor: one for chain data and one for state**

trie:

* get
* contains
* insert
* remove
* root
* get_proof
* verify_proof

### The database

* [db form cita-trie]:

```
/// "DB" defines the "trait" of trie and database interaction.
/// You should first write the data to the cache and write the data
/// to the database in bulk after the end of a set of operations.
```
    - get
    - contains
    - insert
    - remove
    - insert_batch
    - remove_batch
    - flush

**executor should implement this trait.**

* [memoryDb form cita-trie]:

```rust
pub struct MemoryDB {
    // If "light" is true, the data is deleted from the database at the time of submission.
    light: bool,
    storage: Arc<RwLock<HashMap<Vec<u8>, Vec<u8>>>>,
}
```

**just use the hashmap and for test, no need to consider, and it'a example to implement the cita-trie::DB trait**


* [Trie from cita-trie]:

```rust
pub trait Trie<D: DB, H: Hasher> {}
```

**Implement this trait if want a MPT**


[PatriciaTrie from cita-trie]:

```rust
pub struct PatriciaTrie<D, H>
where
    D: DB,
    H: Hasher,
{}
```

**Directly use PatriciaTrie**

## The plan

* Refactor the `cita-executor/core/src/state/mod.rs`: split the state, account, and account_entry.(follow the cita-vm)
* Try to refactor the `super_admin_account` of the state. (Remove it from the structure)
* Check the `account_start_nonce`. (Remove it from the structure)
* ~~Remove uselsee column.~~ (node info. need to keep it for compatibility)
* check the account bloom. (is it must-have for snapshot, or can remove it when rewite)
* Remove the useless db, only use rocksDB(and some caches), (follow cita-trie)
* Analyse process about state and define db interface
* Rewite the rocksdb part.
* Rewite the common-types.
* Use cita-vm(use cita-trie at the same time)
* Clean

[StateDB]: https://github.com/cryptape/cita/blob/develop/cita-executor/core/src/state_db.rs#L89
[State]: https://github.com/cryptape/cita-vm/blob/master/src/state/state.rs#L19
[executor-State]: https://github.com/cryptape/cita/blob/develop/cita-executor/core/src/state/mod.rs#L248
[cita-Column]: https://github.com/cryptape/cita/blob/develop/cita-chain/types/src/db.rs#L26
[db form cita-trie]: https://github.com/cryptape/cita-trie/blob/master/src/db.rs#L12
[memoryDb form cita-trie]: https://github.com/cryptape/cita-trie/blob/master/src/db.rs#L53
[Trie from cita-trie]: https://github.com/cryptape/cita-trie/blob/master/src/trie.rs#L16
[PatriciaTrie from cita-trie]: https://github.com/cryptape/cita-trie/blob/master/src/trie.rs#L52
