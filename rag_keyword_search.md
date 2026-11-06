# RAG 关键词高速检索方法
**概述:**  RAG 系统的最终合规校验环节，针对固定敏感词库进行高效匹配与过滤，采用多模式关键词匹配机制构建候选匹配集合，并结合最长优先、正向最大匹配策略，提高敏感词识别的准确性与匹配效率。

针对 jieba 默认分词规则在包含标点符号、空格等特殊字符的关键词场景下，可能出现词语拆分、完整关键词无法命中的问题，对其词典加载、DAG 构建及分词路径选择等逻辑进行扩展，使自定义关键词能够按照预期保持整体匹配，提升复杂文本场景下的关键词识别能力。该方案也可复用于其他固定词汇的快速检索与文本过滤场景。

## 核心关键
修改jieba的拆分规则。如果不修改，那些固定词就会被jieba分词，达不到作为一个整体词分词的目的。jieba分词有自己的词典，词典配置使用空格作为固定的分割符号。这里要改掉空格作为固定的分隔符分割。使用两个unicode编码为\u0040（就是@字符）作为分割。

## 修改jieba/_init_.py
将这句：

re_userdict = re.compile('^(.+?)( [0-9]+)?( [a-z]+)?$', re.U)

替换为：

re_userdict = re.compile('^(.+?)(\u0040\u0040[0-9]+)?(\u0040\u0040[a-z]+)?$', re.U)

将这句：

re_han_default = re.compile("([\u4E00-\u9FD5a-zA-Z0-9+#&\._%\-]+)", re.U)

替换为：

re_han_default = re.compile("(.+)", re.U)

gen_pfdict方法中将这句：

word, freq = line.split(' ')[:2]

替换为：

word, freq = line.split('\u0040\u0040')[:2]

## 修改词典分割符号保持一致
将jieba/dict.txt辞典的空格替换成@@

修改jieba/_init_.py加载词典方法load_userdict,在

word, freq, tag = re_userdict.match(line).groups() 下面补充一行：

freq = freq.replace('\u0040\u0040', '')

修改jieba/_init_.py加载词典方法add_word,将

self.user_word_tag_tab[word] = tag 替换为：

self.user_word_tag_tab[word] = tag.replace('\u0040\u0040', '')




