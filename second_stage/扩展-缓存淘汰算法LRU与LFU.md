# 缓存淘汰算法LRU与LFU

## 概念

---
常用的缓存过期淘汰策略有：LRU、LFU、FIFO等，因为FIFO（First in first out）算法是一种先进先出的队列，算法很简单，就不具体描述了，下面介绍LRU和LFU的算法策略。

LRU：最近最少使用淘汰算法（Least Recently Used）。如果一个数据在最近一段时间内没有被访问到，那么可以认为在将来它被访问的可能性也很小。因此当空间满时，最久没有被访问的数据最先被淘汰。  
LFU：最不经常使用淘汰算法（Least Frequently Used），如果一个数据在最近一段时间内很少被访问到，那么可以认为在将来它被访问的可能性也很小。因此当空间满时，最小频率访问的数据最先被淘汰。

</br>

## 算法实现

---

**LRU：**  
最常见的算法是使用一个链表保存缓存数据，不考虑时间复杂度的基本算法思想如下：  

1. 新数据插入到链表头。
2. 缓存命中的数据（被访问到的），将移到链表头。
3. 当链表满时，将链表尾部数据淘汰（丢弃）。

leetcode上关于LRU算法的题目：<https://leetcode-cn.com/problems/lru-cache/>  

O(1)时间复杂度的解法：  
【哈希表+双向链表】算法思想：  

定义一个哈希表ket_table，该哈希表的索引为key，每个索引存放链表里的内存地址，链表中存放着缓存数据。  

当执行get(key)操作时，如果该key存在于哈希表中，通过key可以直接得到缓存的内存地址，将值取出返回，如果不存在则返回-1。当查找到缓存时，需要将缓存数据置于链表头部，表示该缓存是最新被访问的。  

当执行put(key, value)操作时，如果key存在于哈希表中，删除链表中的数据，再将新数据插入到链表头部，并将头部的内存地址赋值给该key的哈希表中，如果不存在，则判断是否缓存已满，没有满则直接插入链表和哈希表中，满了则删除链表尾部的缓存数据，然后再进行插入。

``` C++
class LRUCache {
    int m_capacity;
    list<pair<int, int> > lrus;
    unordered_map<int, list<pair<int, int> >::iterator> m_key_table;
public:
    LRUCache(int capacity) {
        m_capacity = capacity;
        m_key_table.clear();
    }

    int get(int key) {
        if (m_capacity == 0) return -1;

        if (m_key_table.count(key)) {
            // 如果key存在，则取出value，并把数据移到链表头
            int value = m_key_table[key]->second;
            lrus.erase(m_key_table[key]);
            lrus.push_front({key, value});
            m_key_table[key] = lrus.begin();
            return value;
        }
        return -1;
    }

    void put(int key, int value) {
        if (m_capacity == 0) return;

        if (m_key_table.count(key)) {
             // 如果存在，则删除链表里的数据
            lrus.erase(m_key_table[key]);
        } else if (m_key_table.size() == m_capacity) {
             // 如果超出了缓存容量，则需要链表尾部删除
            m_key_table.erase(lrus.back().first);
            lrus.pop_back();
        }
        // 新插入的数据放到链表头
        lrus.push_front({key, value});
        m_key_table[key] = lrus.begin();
    }
};
```

</br>

**LFU：**  
leetcode上关于LFU算法的题目：<https://leetcode-cn.com/problems/lfu-cache/>  
O(1)时间复杂度解法：  
【哈希表+双向链表】算法思想：  

该算法与LRU算法类似，不过LFU需要用到访问频率信息，所以该算法需要两个哈希表实现。  

第一个哈希表freq_table的索引为freq（访问频率），每个索引存放一个双向链表，第二个哈希表key_table的索引为key，每个索引存放该key在链表中的内存地址。还需要一个类变量m_min_freq用于保存最小的访问频率，当需要进行删除缓存的时候，可以用该变量定位缓存的位置进行删除。  

当执行get(key)操作时，通过key在key_table中获取缓存的内存地址，如果不存在，则返回-1，如果存在，则取出缓存数据，访问频率需要增加1，所以要将缓存放入freq_table[freq+1]的链表头部。  

当执行put(key, value)操作时，判断该key是否在key_table中，如果不存在，并且容量没有满，则直接插入到freq_table[1]的链表中，如果容量满了，则通过m_min_freq找到最小访问频率的链表，从尾部删除数据，然后将新数据直接插入到freq_table[1]的链表头部。如果key存在与key_table中，则取出链表中的缓存数据，更新新的value，并增加访问频率(freq+1)，将插入到freq_table[1]的链表头部。

``` c++
typedef struct LFUStruct {
    int key, value, freq;
    LFUStruct (int k, int v, int f) : key(k), value(v), freq(f) {}
} lfu;

class LFUCache {
    int m_capacity;
    int m_min_freq;

    unordered_map<int, list<lfu>::iterator> m_key_table;
    unordered_map<int, list<lfu>> m_freq_table;
public:
    LFUCache(int capacity) {
        m_capacity = capacity;
        m_min_freq = 0;
        m_key_table.clear();
        m_freq_table.clear();
    }

    int get(int key) {
        if (m_capacity == 0) return -1;

        auto it = m_key_table.find(key);
        if (it == m_key_table.end()) return -1;

        // 存在key，则取出value并且freq+1，把节点移入到m_freq_table[freq+1]的链表顶部
        auto get_it = it->second;
        int value = get_it->value;
        int freq = get_it->freq;
        m_freq_table[freq].erase(get_it);
        if (m_freq_table[freq].size() == 0) {
            m_freq_table.erase(freq);
            if (m_min_freq == freq) m_min_freq += 1;
        }
        m_freq_table[freq + 1].push_front(lfu(key, value, freq + 1));
        m_key_table[key] = m_freq_table[freq + 1].begin();
        return value;
    }

    void put(int key, int value) {
        if (m_capacity == 0) return;
        auto it = m_key_table.find(key);
        // 如果没有找到该key，表明不存在，则插入
        if (it == m_key_table.end()) {
            // 当缓存达到最大容量时，需要删除访问频率最少的键
            if (m_key_table.size() >= m_capacity) {
                auto del = m_freq_table[m_min_freq].back(); // 链表末尾弹出需要删除的节点
                m_freq_table[m_min_freq].pop_back();
                m_key_table.erase(del.key);
                if(m_freq_table[m_min_freq].size() == 0) {
                    m_freq_table.erase(m_min_freq);
                }
            }
            // 插入新缓存
            m_freq_table[1].push_front(lfu(key, value, 1));
            m_key_table[key] = m_freq_table[1].begin();
            m_min_freq = 1;
        } else {
            //如果找到key，则更新
            auto update_it = it->second;
            int freq = update_it->freq;
            m_freq_table[freq].erase(update_it); // 删除freq_table里的缓存节点
            if (m_freq_table[freq].size() == 0) {
                m_freq_table.erase(freq);
                if (m_min_freq == freq) {
                    m_min_freq += 1;
                }
            }
            m_freq_table[freq + 1].push_front(lfu(key, value, freq + 1));
            m_key_table[key] = m_freq_table[freq + 1].begin();
        }
    }
};
```
