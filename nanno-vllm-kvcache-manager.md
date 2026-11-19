# nano-vllm 的 KV Cache 分页内存管理机制
**概述:**  nano-vLLM 的 BlockManager 是 PagedAttention 机制中 KV Cache 分页内存管理的核心组件，负责 GPU KV Cache 物理 Block 的分配、释放与复用。通过维护 Sequence 的 block_table，建立逻辑 Block 与物理 Block 之间的映射，实现 KV Cache 的非连续存储与动态管理。同时，结合 Block Hash、引用计数等机制实现 Prefix Caching，使多个请求能够共享相同前缀的 KV Cache，避免重复计算前缀的 K、V，减少 Prefill 阶段的计算开销，并降低 KV Cache 的重复存储需求，提高 GPU 显存利用率和推理效率。

## 核心结构
### Block块结构
这个Block块是物理块信息，包括块的ID，这个块相应的token列表。核心结构如下：
```python
class Block:

    def __init__(self, block_id):
        # block_id 物理块ID
        self.block_id = block_id
        # 引用计数
        self.ref_count = 0
        # hash，这个是prefix cache要用到的，从第一个提示词到当前token的所有的hash值
        self.hash = -1
        # 当前块的token id 列表
        self.token_ids = []

    def update(self, hash: int, token_ids: list[int]):
        self.hash = hash
        self.token_ids = token_ids

    # 用于申请块成功后做个初始化
    def reset(self):
        self.ref_count = 1
        self.hash = -1
        self.token_ids = []
```
### Sequence结构
Sequence描述的是每个请求当作一个序列管理，核心结构如下：
```python
class Sequence:
    block_size = 256
    counter = count()

    def __init__(self, token_ids: list[int], sampling_params = SamplingParams()):
        self.seq_id = next(Sequence.counter)
        self.status = SequenceStatus.WAITING
        self.token_ids = copy(token_ids)
        self.last_token = token_ids[-1]
        self.num_tokens = len(self.token_ids)
        self.num_prompt_tokens = len(token_ids)
        self.num_cached_tokens = 0
        self.num_scheduled_tokens = 0
        # 请求进来肯定是先进行prefill
        self.is_prefill = True
        # block_table 就是pageattenion模拟的叶管理机制，里面存的是真实的物理块ID，block_table列表的下标就是逻辑块ID，就对应起来了。block_table列表有顺序的，通过block_table的列表的顺序遍历就可以计算前缀等信息
        self.block_table = []
```

### BlockManager结构
BlockManager就是管理Block块的对象，管理的是真实的显存块。
```python
class BlockManager:

    def __init__(self, num_blocks: int, block_size: int):
        # block_size 记录每个块包含多少token
        self.block_size = block_size
        # num_blocks：是显存可以分为多少块，初始化时候已经计算好
        # blocks：这个就是初始化，Block里面的block_id和blocks下标索引是一致的
        self.blocks: list[Block] = [Block(i) for i in range(num_blocks)]
        # hash_to_block_id：实现prefix技术使用的字典,记录哪些物理ID可以共用
        self.hash_to_block_id: dict[int, int] = dict()
        # free_block_ids：初始化是所有num_blocks的编号
        self.free_block_ids: deque[int] = deque(range(num_blocks))
        # used_block_ids：已经使用记录到集合
        self.used_block_ids: set[int] = set()
```
#### BlockManager核心方法解析
_allocate_block：分配块
```python
def _allocate_block(self) -> int:
    # 从双端队列free_block_ids的左边取出一个索引，这个索引就是block_id
    block_id = self.free_block_ids.popleft()
    # 列表blocks的下标也就是block_id
    block = self.blocks[block_id]
    assert block.ref_count == 0
    # 因为free_block_ids 会放入已经回收的块，但是hash表的值还是旧值，需要清除
    if block.hash != -1 and self.hash_to_block_id.get(block.hash) == block_id:
        del self.hash_to_block_id[block.hash]
    # 申请成功后做初始化
    block.reset()
    # used_block_ids 记录已经使用的块ID
    self.used_block_ids.add(block_id)
    return block_id
```

_deallocate_block：回收块
```python
def _deallocate_block(self, block_id: int):
    # 引用技术必须是0
    assert self.blocks[block_id].ref_count == 0
    # 从used_block_ids移除块ID
    self.used_block_ids.remove(block_id)
    # 放回到free_block_ids的尾部
    self.free_block_ids.append(block_id)
```

can_allocate：做个预计算，计算是否可以申请块，如果没有足够的块返回-1，如果还有，返回有多少相同的前缀块。
```python
def can_allocate(self, seq: Sequence) -> int:
    h = -1
    num_cached_blocks = 0
    # seq.num_blocks 是根据token数量计算得到需要多少个block块的
    num_new_blocks = seq.num_blocks
    # 每次都要从头开始算起，因为prefix是链式hash
    for i in range(seq.num_blocks - 1):
        token_ids = seq.block(i)
        h = self.compute_hash(token_ids, h)
        block_id = self.hash_to_block_id.get(h, -1)
        # 如果block_id==-1，表示找不到，就说明从这里分叉了
        # self.blocks[block_id].token_ids != token_ids，表示这个块里面的token不一样，也要从这里分叉了
        if block_id == -1 or self.blocks[block_id].token_ids != token_ids:
            break
        # 记录有多少个相同的块
        num_cached_blocks += 1
        # 表示已经缓存在KVcache里面了，可以共用，不需要申请
        if block_id in self.used_block_ids:
            num_new_blocks -= 1
    # 剩余的比需要的还少，没有空间可以申请了，返回-1
    if len(self.free_block_ids) < num_new_blocks:
        return -1
    
    # 返回有共享多少个块，这个也可以是0
    return num_cached_blocks
```

allocate：真正分配块的操作，这个与can_allocate有所不同
```python
def allocate(self, seq: Sequence, num_cached_blocks: int):
    assert not seq.block_table
    h = -1
    # num_cached_blocks 是已经计算的有prefix的块
    for i in range(num_cached_blocks):
        token_ids = seq.block(i)
        h = self.compute_hash(token_ids, h)
        block_id = self.hash_to_block_id[h]
        block = self.blocks[block_id]
        # 这个块可能在已经使用的列表里
        if block_id in self.used_block_ids:
            block.ref_count += 1
        else: # 也可能不在使用的列表里，空闲了但保留了hash缓存映射
            block.ref_count = 1
            self.free_block_ids.remove(block_id)
            self.used_block_ids.add(block_id)
        
        # 追加到sequence的block_table尾部，要记录所有的block_id
        seq.block_table.append(block_id)

    # 剩余的分叉，没有前缀的，继续补充
    for i in range(num_cached_blocks, seq.num_blocks):
        seq.block_table.append(self._allocate_block())
    seq.num_cached_tokens = num_cached_blocks * self.block_size
```
