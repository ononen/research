# TESTS

CITA-TESTS

## What we have

除了分布在代码里的单元测试，其余测试代码主要在 `tests/` 目录下，主要包括以下部分：

* box_executor
    mock consensus 通过 MQ 发 SignedProposal/BlockWithProof 消息的方式测试交易预执行是否有效
* chain_performance_by_mq
    mock block 通过 MQ 发 Block 消息对 Chain 模块进行性能(压力)测试
* chain_executor_mock
    mock chain executor 的数据
* compatibility
    检查 genesis 是否有改变
* consensus-mock
    mock consensus 的数据
* integrate_test
    - basic
    - crosschain
    - robustness
    - byzantine
    - charge mode
    - fee back
* interfaces
    rpc 接口的测试

## What we want

* 单元测试
* 基本功能测试，形成独立的测试框架及标准一致的测试用例
* 集成测试
* 内部模块测试
    - executor
    - evm（使用 ethereum-tests 相关测试用例）
    - chain
    - network
    - auth
    - jsonrpc
    - (bft)
* byzantine
    - fuzz
* compatibility

## How

* 短期： 整理现有测试
* 长期： 逐步形成测试框架及用例

## Resource

[cita tests](https://github.com/cryptape/cita/tree/develop/tests)
[ethereum_tests](https://github.com/ethereum/tests)
[bitcoin test](https://github.com/bitcoin/bitcoin/tree/master/test)
[json schema](https://github.com/json-schema-org/json-schema-spec)
