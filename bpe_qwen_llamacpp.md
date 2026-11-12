# BPE 分词算法机制剖析
**概述:**  BPE（Byte Pair Encoding）是一种基于子词合并的分词算法，通过统计语料中高频相邻字符对并按优先级迭代合并，构建词表，实现文本到 Token ID 的高效映射。

基于 llama.cpp 源码，深入解析 BPE 分词的核心实现，梳理词表、合并规则等关键数据结构，剖析相邻 Token 的优先级匹配与迭代合并逻辑，完整梳理从提示词文本预处理、子词合并到 Token ID 生成的处理流程。

## 数据文件
BPE 分词有两个文件，分别是词典文件vocab.json，是一个json结构，存的是字符和ID的映射关系。另外一个是合并规则文件merges.txt，存的字符的合并规则。

## 分词逻辑
### 先将提示词prompt经过正则表达式切分成词
每个大模型都有自己的正则表达式切分规则,先按照正则做切分得到词列表。

```cpp
const auto word_collection = unicode_regex_split(text, regex_exprs, true);
```

### 将对每个词开始处理，记录每个词里面字符的相关信息
核心结构如下
```cpp
// llm_symbol记录字符相关信息
struct llm_symbol {
    // prev，指向当前字符所在索引的前一个索引
    // next，指向当前字符所在索引的下一个索引
    // text，表示指向当前字符的首地址
    // n 表示 当前字符占用大小，为后续从text获取字符提供范围
    using index = int;
    index prev;
    index next;
    const char * text;
    size_t n;
};
```
处理逻辑：对word_collection每个word遍历，记录word里面每个字符的前驱，后继等信息
```cpp
// symbols 存储正则分词后的单个词的字符信息，当前指针，大小，前驱，后继
std::vector<llm_symbol> symbols;
int index = 0;
size_t offset = 0;
while (offset < word.size()) {
    llm_symbol sym;
    // n 表示 当前字符大小
    // text，表示指向当前字符的首地址
    // prev，指向当前字符所在索引的前一个索引
    // next，指向当前字符所在索引的下一个索引
    size_t char_len = std::min(word.size() - offset, (size_t) unicode_len_utf8(word[offset]));
    sym.text = word.c_str() + offset;
    sym.n = char_len;
    offset += sym.n;
    sym.prev = index - 1;
    sym.next = offset == word.size() ? -1 : index + 1;
    index++;
    symbols.emplace_back(sym);
}
```

### 创建队列
有个关键的数据结构，使用优先队列，根据merges.txt提供数据，越靠前的优先合并。
```cpp
template<typename T, typename Container = std::vector<T>, typename Compare = std::less<typename Container::value_type>>
class llama_priority_queue : public std::priority_queue<T, Container, Compare> {
public:
    using std::priority_queue<T, Container, Compare>::priority_queue;

    T pop_move() {
        T item = std::move(this->c.front());
        std::pop_heap(this->c.begin(), this->c.end(), this->comp);
        this->c.pop_back();
        return item;
    }

    void pop() = delete;
};

struct llm_bigram_bpe {
    struct comparator {
        bool operator()(const llm_bigram_bpe & l, const llm_bigram_bpe & r) const {
            return l.rank > r.rank || (l.rank == r.rank && l.left > r.left);
        }
    };

    using queue_storage = std::vector<llm_bigram_bpe>;
    using queue = llama_priority_queue<llm_bigram_bpe, queue_storage, comparator>;
    // left ， right是vector<llm_symbol>的下标
    // text , 表示left 和right下标对应的字符可以合并，拼接在一起的字符
    // 在合并规则表的排序，越靠前越优先合并
    // rank，加载merges.txt的顺序，从1开始递增
    // size ，可以合并字符后的大小
    llm_symbol::index left;
    llm_symbol::index right;
    std::string text;
    int rank;
    size_t size;
};
```
### 进行合并
合并之前，需要初始化将std::vector<llm_symbol> symbols里面的数据相邻的构建组合。
```cpp
for (int i = 1; i < (int) symbols.size(); ++i) {
    // 先将相邻的两个字符构造一对
    add_new_bigram(i - 1, i);
}
// 将相邻的两个字符合并，判断是否在mergers.txt中存在
llm_bigram_bpe::queue work_queue;
void add_new_bigram(int left, int right) {
    if (left == -1 || right == -1) {
        return;
    }

    std::string left_token  = std::string(symbols[left].text,  symbols[left].n);
    std::string right_token = std::string(symbols[right].text, symbols[right].n);

    const int rank_found = vocab.find_bpe_rank(left_token, right_token);
    if (rank_found < 0) {
        return;
    }

    llm_bigram_bpe bigram;
    bigram.left  = left;
    bigram.right = right;
    bigram.text  = left_token + right_token;
    bigram.size  = left_token.size() + right_token.size();
    bigram.rank  = rank_found;

    work_queue.push(bigram);
}
```
开始合并
```cpp
//合并符合在merges.txt规则的字符对
while (!work_queue.empty()) {
    auto bigram = work_queue.pop_move();

    auto & left_symbol  = symbols[bigram.left];
    auto & right_symbol = symbols[bigram.right];

    if (left_symbol.n == 0 || right_symbol.n == 0) {
        continue;
    }

    std::string left_token  = std::string(left_symbol.text,  left_symbol.n);
    std::string right_token = std::string(right_symbol.text, right_symbol.n);

    std::cout<<"left_token:"<<left_token<<",right_token:"<<right_token<<std::endl;

    if (left_token + right_token != bigram.text) {
        continue;
    }

    // 可以合并，取元素的时候是根据地址和长度获取
    left_symbol.n += right_symbol.n;

    // n设置0，表示该元素被合并了
    right_symbol.n = 0;

    // 修改指向，类似双向链表删除某个元素时后继和前驱指针需要修改
    left_symbol.next = right_symbol.next;
    if (right_symbol.next >= 0) {
        symbols[right_symbol.next].prev = bigram.left;
    }

    // 合成新的一个元素后，新的元素和他前面的元素要继续判断是否可以合并
    add_new_bigram(left_symbol.prev, bigram.left);

    // 新的元素也要和他后面的元素继续判断是否可以合并
    add_new_bigram(bigram.left, left_symbol.next);
}
```
### 合并结束后，就可以输出
由于合并中，某个字符的n已经设置为0，这部分过滤掉即可。
```cpp
using bpe_token = int32_t;
std::vector<bpe_token> & output;
for (const auto & symbol : symbols) {
    if (symbol.n == 0) {
        continue;
    }

    //截取字符串
    const std::string str = std::string(symbol.text, symbol.n);
    //查表，词典vocab.json经过json解析后放入到表里，查表就可以得到 TOKEN ID
    const auto token = vocab.text_to_token(str);
    output.push_back(token);
}
```


