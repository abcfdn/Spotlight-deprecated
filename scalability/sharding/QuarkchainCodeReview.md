# An Overview On Quarkchain Codebase 

__By ABC Blockchain Community__

Author: __Yijie Hong, Haoran Qi, Yan Zhu, Shu Dong__


## About ABC Blockchain Community

ABC Blockchain Community is a technical community for blockchain enthusiasts to learn, apply and explore the blockchain technology. Learn more at [our website](www.abcer.world). 


## About Quarkchain

QuarkChain is a high-throughput blockchain that aims to achieve millions of transactions per second (TPS) through use of sharding. Quarkchain release their test-net on Sep 17th 2018 and open sourced their code base. 

This review is done on Oct 30th against version [2d4f43f54523e6c1768432690cabe496aa860107](https://github.com/QuarkChain/pyquarkchain/tree/2d4f43f54523e6c1768432690cabe496aa860107/quarkchain).


##Why Code Review

There are many writings on blockchain projects’ innovative ideas and sparking business plans, but not enough professional review on the engineering quality of a project. However the quality of the project codebase is important in the open source world as it directly affect future developer contributions and the eco-system built-up. This article offers an overview for Quarkchain’s code structure, contribution activeness and alignance with their white paper. It can also be used as a guide if you plan to contribute to their codebase. 

###Code Structure

[Repo Link](https://github.com/QuarkChain/pyquarkchain/tree/)

 - devp2p: forked from [pydevp2p](https://github.com/ethereum/pydevp2p/tree/develop/devp2p)
 - quarkchain/p2p/: forked from [Trinity](https://github.com/ethereum/py-evm/tree/master/p2p)
 - quarkchain/evm/ & transaction processing: forked from [pyethereum](https://github.com/ethereum/pyethereum/tree/develop/ethereum)
 - ethereum/pow/: forked from [pyethereum](https://github.com/ethereum/pyethereum/tree/develop/ethereum/pow)
 - quarkchain/cluster/: most of the sharding related code located here

For code forked from ethereum, Quarkchain made some customizations to fit their sharding design and added a significant amount of tests to ensure compatibility. One example is the [account](https://github.com/QuarkChain/pyquarkchain/blob/2d4f43f54523e6c1768432690cabe496aa860107/quarkchain/accounts.py#L40) and [address](https://github.com/QuarkChain/pyquarkchain/blob/2d4f43f54523e6c1768432690cabe496aa860107/quarkchain/core.py#L325) Model: 4 additional bytes is added to the account address as shard ID, extending which from 20 bytes to 24 bytes. 4 new fields are also added to the [transaction](https://github.com/QuarkChain/pyquarkchain/blob/2d4f43f54523e6c1768432690cabe496aa860107/quarkchain/evm/transactions.py#L61-L64) model and [transaction validation](https://github.com/QuarkChain/pyquarkchain/blob/2d4f43f54523e6c1768432690cabe496aa860107/quarkchain/cluster/shard_state.py#L193) logic is thus adjusted accordingly.

###Repo State:

The repo is being actively maintained.
 
 - 15 branches
 - 1024 commits
 - 9 contributors

Additional thumb-ups from our reviewers: 
 
 - Last commit made hours ago, people are actively working on it.
 - Issue/Pull Request is being used.
 - Wiki is well maintained

###Code style:

Additional thumb-ups from our reviewers: 
 
 - Clear comments
 - Good function naming
 - Sufficient and clear tests for corner cases/attacks: [Example](https://github.com/QuarkChain/pyquarkchain/blob/2d4f43f54523e6c1768432690cabe496aa860107/quarkchain/cluster/tests/test_root_state.py#L371)
 - Good abstraction of component: E.g. db module is well abstracted, which supports multiple dbs ([code](https://github.com/QuarkChain/pyquarkchain/blob/master/quarkchain/db.py)).

###Sharding

Comparing to other blockchain sharding projects, Quarkchain adopts the concept of "clustering" (inspired by [Google's Bigtable](https://en.wikipedia.org/wiki/Bigtable)) in design, where each cluster(full node) consists of 1 master node and many slave nodes. Every cluster(full node) contains all information in the blockchain network. In each cluster, one shard is only handled by one slave node, but one slave node can handle multiple shards. When new block comes, master node will assign the block to a slave node. Such design simplifies the routing between different shards(via master nodes) but also makes master node a single point failure in the cluster, though such failure won't affect the security of the whole network thanks to replica. Such design also allows light nodes become a cluster and operates as a full node which helps ensure the decentralization of the blockchain.

Each cluster, which is equivalent to a full node, consists of [one master server](https://github.com/QuarkChain/pyquarkchain/blob/2d4f43f54523e6c1768432690cabe496aa860107/quarkchain/cluster/master.py#L531) and a bunch of slave servers. Masters mine for the root blocks. Also, they manage the connections to all slaves within the cluster through a slave_pool. The initialization of a master setups both master-slaves connections and slave-slave connections. [One slave server](https://github.com/QuarkChain/pyquarkchain/blob/2d4f43f54523e6c1768432690cabe496aa860107/quarkchain/cluster/slave.py#L777) can run one or more branches, one shard per branch. Slave mines for minor blocks, which can run different consensus protocol. The difficulty of mining will be adjusted when the minor block headers are included in the root block. <span style="color:red"> ___The root block validates the header and difficulties of minor blocks, but not every transaction___ </span>.

The ledger, similar to the chain of blocks in Bitcoin and Ethereum, is “sharded” into root blocks and minor blocks. The [shard chain](https://github.com/QuarkChain/pyquarkchain/blob/2d4f43f54523e6c1768432690cabe496aa860107/quarkchain/cluster/shard_state.py)(containing [minor blocks](https://github.com/QuarkChain/pyquarkchain/blob/2d4f43f54523e6c1768432690cabe496aa860107/quarkchain/core.py#L723)) record the EVM states of the shard, gas used within and cross shards([check how cross shard gas cost is calculated here](https://github.com/QuarkChain/pyquarkchain/blob/2d4f43f54523e6c1768432690cabe496aa860107/quarkchain/cluster/shard_state.py#L858-L863)), and transactions being handled in this shard. They also calculates a hash for the validation of this block, which is included in the minor block header(check add_block & validate_block function). The [root chain](https://github.com/QuarkChain/pyquarkchain/blob/2d4f43f54523e6c1768432690cabe496aa860107/quarkchain/cluster/root_state.py)(containing [root blocks](https://github.com/QuarkChain/pyquarkchain/blob/2d4f43f54523e6c1768432690cabe496aa860107/quarkchain/core.py#L938)) collect the headers of all minor blocks with validation based on the hash. The root blocks maintain root status, which does not include a specific transaction, but hash of all shards well as shards informations(check [add_block](https://github.com/QuarkChain/pyquarkchain/blob/2d4f43f54523e6c1768432690cabe496aa860107/quarkchain/cluster/root_state.py#L392) and [validate_block](https://github.com/QuarkChain/pyquarkchain/blob/2d4f43f54523e6c1768432690cabe496aa860107/quarkchain/cluster/root_state.py#L277) function). This implementation enables the split of ledger among shards, so that the capacity of overall ledger can increase linearly as the number of shards increase but the capacity of each node does not change. The previous conclusion is based on the assumption that the transactions are distributed evenly among shards. To distribute the computation power evenly between root chain and shard chain, the concept “tax” is applied to the system. For each minor block, the miner will only get half of the block reward([Code](https://github.com/QuarkChain/pyquarkchain/blob/2d4f43f54523e6c1768432690cabe496aa860107/quarkchain/cluster/shard_state.py#L560-L561)) while the rest, as the tax, is paid to miners in root chain([Code](https://github.com/QuarkChain/pyquarkchain/blob/2d4f43f54523e6c1768432690cabe496aa860107/quarkchain/cluster/root_state.py#L205-L209)).

### Suggestions for future work
1. According to [their wiki](https://github.com/QuarkChain/pyquarkchain/wiki/Smart-Contract), cross shard smart contract is not supported, only cross-shard QKC transfering is supported. In current version, to run a smart contract in a shard, if users don’t have any QKC in this shard, they have to transfer enough QKC to the shard via cross shard transfering first.

2. For now there is no clear way to add shards dynamically. As the network grows we need to set up the mechanism eventually.

3. Tax rate is hardcoded. One area of improvement could be adjustable tax according to current state, to handle cases when we need more than 50% of computation power in root chain to ensure the security.
