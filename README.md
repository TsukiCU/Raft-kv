# raft-kv

结构分析（混乱版）

Everything is all mixed up and I definitely need to find a time to reorganize my thoughts and rewrite the readme from the ground up.


## Test (using redis-cli)

    redis-cli -p 63791
    127.0.0.1:63791> set mykey myvalue
    OK
    127.0.0.1:63791> get mykey
    "myvalue"
    
remove a node and replace the myvalue with "new-value" to check cluster availability:

    goreman run stop node2
    redis-cli -p 63791
    127.0.0.1:63791> set mykey new-value
    OK
    
bring the node back up and verify it recovers with the updated value "new-value":

    redis-cli -p 63792
    127.0.0.1:63792> KEYS *
    1) "mykey"
    127.0.0.1:63792> get mykey
    "new-value"
    
### benchmark

    redis-benchmark -t set,get -n 100000 -p 63791
    
    ====== SET ======
      100000 requests completed in 1.35 seconds
      50 parallel clients
      3 bytes payload
      keep alive: 1
    
    96.64% <= 1 milliseconds
    99.15% <= 2 milliseconds
    99.90% <= 3 milliseconds
    100.00% <= 3 milliseconds
    73909.83 requests per second
    
    ====== GET ======
      100000 requests completed in 0.95 seconds
      50 parallel clients
      3 bytes payload
      keep alive: 1
    
    99.95% <= 4 milliseconds
    100.00% <= 4 milliseconds
    105485.23 requests per second

## 整体流程 （以用户端输入一条set指令为例）

用户通过 redis-benchmark 工具向服务器发送 SET 命令。整个流程是：redis-benchmark → 服务器上的RedisSession对象开始处理这个命令 -> RedisSession::handle_read (使用hiredis进行解析) → 解析完成，得到redisReply* reply → 查看RedisSession::command_table 得到回调函数shared::CommandCallback cb (set对应的是set_command) → RedisSession::set_command → RedisStrore::set (RedisSession类持有RedisStore* server_) → 构建 RaftCommit对象 commit, 序列化 → RaftNode::propose(RedisStore类持有RaftNode* server_, 注意和RedisSession中的server_不同) → Node::propose (RaftNode类持有std::unique_ptr<Node> node_) → 定义proto::MessagePtr msg，初始化type, from, entries → Raft::step (Node类持有RaftPtr raft_) → …(Raft协议的实现细节)

线程模型：RaftNode 中的操作需要在特定的线程中执行，以保证线程安全和正确性。

异步处理：整个流程中大量使用异步回调和事件驱动机制，避免阻塞，提高性能。

数据一致性：通过 Raft 协议，确保写操作在集群中的一致性，即使有节点故障，也能保证数据的正确性。

持久化存储：使用 RocksDB 作为底层存储引擎，提供高性能的持久化存储。

错误处理：在每个步骤中，都有完善的错误处理和日志记录，确保问题可以被及时发现和解决。


## 核心组件分析

### Raft库
Raft库屏蔽了网络、存储等模块，提供接口由上层应用者来实现。

#### enum RaftState
节点一共有四种状态.
```
enum RaftState {
  Follower = 0,
  Candidate = 1,
  Leader = 2,
  PreCandidate = 3,
};
```

#### struct Ready
Ready类中包括：SoftStatePtr, HardStatePtr (这两者的区分是是否需要写入WAL中。SoftState存储易变且不需要保存在WAL日志中的状态数据，包括：集群leader、节点的当前状态。HardState包括节点当前Term、Vote、Commit等。） ReadStates(用于一致性读, Snapshot (需要写入持久化存储中的快照数据), CommittedEntries(需要输入到状态机中的数据，这些数据之前已经被保存到持久化存储中了), Messages(在entries被写入持久化存储中以后，需要发送出去的数据), Entries(在向其他集群发送消息之前需要先写入持久化存储的日志数据).

```RaftNode```

RaftNode
    
    
    
    

