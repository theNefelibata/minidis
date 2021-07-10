# minidis
简易key-value存储引擎实现
# skiplist

Created: July 1, 2021 11:29 AM
Created By: hu yi
Last Edited By: hu yi
Last Edited Time: July 10, 2021 9:38 PM

### 什么是跳表

跳表全称为跳跃列表，它允许快速查询，插入和删除一个有序连续元素的数据链表。跳跃列表的平均查找和插入时间复杂度都是O(logn)。快速查询是通过维护一个多层次的链表，且每一层链表中的元素是前一层链表元素的子集。一开始时，算法在最稀疏的层次进行搜索，直至需要查找的元素在该层两个相邻的元素中间。这时，算法将跳转到下一个层次，重复刚才的搜索，直到找到需要查找的元素为止。

![skiplist%207d766a7ca7c84d558da0141d47c6ced8/Untitled.png](skiplist%207d766a7ca7c84d558da0141d47c6ced8/Untitled.png)

### 跳表的查询、插入与删除

- 查询

![skiplist%207d766a7ca7c84d558da0141d47c6ced8/Untitled%201.png](skiplist%207d766a7ca7c84d558da0141d47c6ced8/Untitled%201.png)

- 插入

首先找到要插入的位置，然后将对应元素插入最下层即可，但是会有一个问题：

如果我们不停的向跳表中插入元素，就可能会造成两个索引点之间的结点过多的情况。结点过多的话，我们建立索引的优势也就没有了。所以我们需要维护索引与原始链表的大小平衡，也就是结点增多了，索引也相应增加，避免出现两个索引之间结点过多的情况，查找效率降低。

跳表是通过一个随机函数来维护这个平衡的，当我们向跳表中插入数据的的时候，我们可以选择同时把这个数据插入到索引里，那我们插入到哪一级的索引呢，这就需要随机函数，来决定我们插入到哪一级的索引中。

这样可以很有效的防止跳表退化，而造成效率变低。

![skiplist%207d766a7ca7c84d558da0141d47c6ced8/Untitled%202.png](skiplist%207d766a7ca7c84d558da0141d47c6ced8/Untitled%202.png)

- 删除

删除操作的话，如果这个结点在索引中也有出现，我们除了要删除原始链表中的结点，还要删除索引中的。因为单链表中的删除操作需要拿到要删除结点的前驱结点，然后通过指针操作完成删除。所以在查找要删除的结点的时候，一定要获取前驱结点。当然，如果我们用的是双向链表，就不需要考虑这个问题了。

### 跳表的实现

跳表节点类

```cpp
template<typename K, typename V>
class Node{

public:
    Node(){}
    Node(K k, V v, int);
    ~Node();

    K get_key() const;
    V get_value() const;
    void set_value(V);

    //保存下一个节点的指针的数组，每层一个，大小为跳表的最大层数
    Node<K, V> **forward;

    int node_level;

private:
    K key;
    V value;
};

template<typename K, typename V>
Node<K,V>::Node(const K k, const V v, int level){
    this->key = k;
    this->value = v;
    this->node_level = level;

    this->forward = new Node<K, V>*[level+1];
    memset(this->forward, 0, sizeof(Node<K,V>*)*(level+1));
}

template<typename K, typename V>
Node<K,V>::~Node() {
    delete []forward;
}

template<typename K, typename V>
Node<K,V>::get_key() const{
    return this->key;
}

template<typename K, typename V>
V Node<K,V>::get_value() const {
    return this->value;
}

template<typename K, typename V>
void Node<K,V>::set_value(V value) {
    this->value = value;
}
```

跳表类的构造与析构

```cpp
template<typename K, typename V>
SkipList<K,V>::SkipList(int max_level) {
    this->_max_level = max_level;
    this->_element_count = 0;
    this->_skip_list_level = 0;

    //创建头节点，kv初始化为null
    K k;
    V v;
    this->_header = new Node<K,V>(k, v, _max_level);
}

template<typename K, typename V>
SkipList<K,V>::~SkipList(){
    if (_file_writer.is_open()) {
        _file_writer.close();
    }
    if (_file_reader.is_open()) {
        _file_reader.close();
    }
    delete _header;
}
```

查询操作

```cpp
// Search for element in skip list
/*
                           +------------+
                           |  select 60 |
                           +------------+
level 4     +-->1+                                                      100
                 |
                 |
level 3         1+-------->10+------------------>50+           70       100
                                                   |
                                                   |
level 2         1          10         30         50|           70       100
                                                   |
                                                   |
level 1         1    4     10         30         50|           70       100
                                                   |
                                                   |
level 0         1    4   9 10         30   40    50+-->60      70       100
*/

template<typename K, typename V>
bool SkipList<K,V>::search_element(K key) {
    std::cout<<"Searching element---------"<<std::endl;
    Node<K,V>* current = _header;

    for(int i = _skip_list_level; i >= 0; i--){
        while(current->forward[i] && current->forward[i]->get_key() < key){
            current = current->forward[i];
        }
    }

    current = current->forward[0];

    if(current && current->get_key() == key){
        std::cout<<"Find key:"<<current->get_key()<<", value:"<<current->get_value()<<std::endl;
        return true;
    }

    std::cout << "Not Found Key:" << key << std::endl;
    return false;
}
```

删除操作

```cpp
// Delete element from skip list
template<typename K, typename V>
void SkipList<K,V>::delete_element(K key){
    mtx.lock();
    Node<K,V>* current = _header;
    Node<K,V>* update[_max_level + 1];
    memset(update, 0, sizeof(Node<K,V>*)*(_max_level + 1));

    for(int i = _skip_list_level; i >= 0; i--){
        while(current->forward[i] && current->forward[i]->get_key() < key){
            current = current->forward[i];
        }
        update[i] = current;
    }
    current = current->forward[0];

    if(current && current->get_key() == key) {

        // start for lowest level and delete the current node of each level
        for (int i = 0; i < _skip_list_level; i++) {

            // if at level i, next node is not target node, break the loop.
            if (update[i]->forward[i] != current[i]) {
                break;
            }
            update[i]->forward[i] = current->forward[i];
        }

        // Remove levels which have no elements
        while (_skip_list_level > 0 && _header->forward[_skip_list_level] == 0) {
            _skip_list_level--;
        }
        std::cout << "Successfully deleted key " << key << std::endl;
        _element_count--;
    }
    mtx.unlock();
}
```

插入操作

```cpp
// Insert given key and value in skip list
// return 1 means element exists
// return 0 means insert successfully
/*
                           +------------+
                           |  insert 50 |
                           +------------+
level 4     +-->1+                                                      100
                 |
                 |                      insert +----+
level 3         1+-------->10+---------------> | 50 |          70       100
                                               |    |
                                               |    |
level 2         1          10         30       | 50 |          70       100
                                               |    |
                                               |    |
level 1         1    4     10         30       | 50 |          70       100
                                               |    |
                                               |    |
level 0         1    4   9 10         30   40  | 50 |  60      70       100
                                               +----+
*/
template<typename K, typename V>
int SkipList<K,V>::insert_element(const K key, const V value){
    mtx.lock();
    Node<K, V> *current = this->_header;

    // create update array and initialize it
    // update is array which put node that the node->forward[i] should be operated later
    Node<K, V> *update[_max_level+1];
    memset(update, 0, sizeof(Node<K, V>*)*(_max_level+1));

    // start form highest level of skip list
    for(int i = _skip_list_level; i >= 0; i--) {
        while(current->forward[i] != NULL && current->forward[i]->get_key() < key) {
            current = current->forward[i];
        }
        update[i] = current;
    }

    // reached level 0 and forward pointer to right node, which is desired to insert key.
    current = current->forward[0];

    // if current node have key equal to searched key, we get it
    if (current != NULL && current->get_key() == key) {
        std::cout << "key: " << key << ", exists" << std::endl;
        mtx.unlock();
        return 1;
    }

    // if current is NULL that means we have reached to end of the level
    // if current's key is not equal to key that means we have to insert node between update[0] and current node
    if(current == NULL || current->get_key() != key){

        // Generate a random level for node
        int random_level = get_random_level();

        // If random level is greater thar skip list's current level, initialize update value with pointer to header
        if(random_level > _skip_list_level){
            for(int i = _skip_list_level+1; i < random_level+1; i++){
                update[i] = _header;
            }
            _skip_list_level = random_level;
        }

        //create new node with random level generated
        Node<K,V>* insert_node = create_node(key, value, random_level);

        for (int i = 0; i < random_level; ++i) {
            insert_node->forward[i] = update[i]->forward[i];
            update[i]->forward[i] = insert_node;
        }
        std::cout << "Successfully inserted key:" << key << ", value:" << value << std::endl;
        _element_count ++;
    }
    mtx.unlock();
    return 0;
}
```

其他

- get_random_level()：插入操作时，通过此函数决定将插入的节点插入到哪一层

    ```cpp
    template<typename K, typename V>
    int SkipList<K, V>::get_random_level(){

        int k = 1;
        while (rand() % 2) {
            k++;
        }
        k = (k < _max_level) ? k : _max_level;
        return k;
    };
    ```

- create_node()：创建一个新节点

    ```cpp
    // create new node 
    template<typename K, typename V>
    Node<K, V>* SkipList<K, V>::create_node(const K k, const V v, int level) {
        Node<K, V> *n = new Node<K, V>(k, v, level);
        return n;
    }
    ```

### 测试：

```cpp
int main(){
    SkipList<std::string, std::string> skipList(6);
    skipList.insert_element("1", "a");
    skipList.insert_element("3", "bc");
    skipList.insert_element("7", "de");
    skipList.insert_element("8", "def");
    skipList.insert_element("9", "g");
    skipList.insert_element("19", "hijk");
    skipList.insert_element("19", "l");

    std::cout << "skipList size:" << skipList.size() << std::endl;

    //skipList.dump_file();

    // skipList.load_file();

    skipList.search_element("9");
    skipList.search_element("18");

    skipList.display_list();

    skipList.delete_element("3");
    skipList.delete_element("7");

    std::cout << "skipList size:" << skipList.size() << std::endl;

    skipList.display_list();
    return 0;
}
```

输出：

```cpp
Successfully inserted key:1, value:a
Successfully inserted key:3, value:bc
Successfully inserted key:7, value:de
Successfully inserted key:8, value:def
Successfully inserted key:9, value:g
Successfully inserted key:19, value:hijk
key: 19, exists
skipList size:6
Searching element---------
Find key:9, value:g
Searching element---------
Not Found Key:18

*****Skip List*****
Level 0:  : ; 1 : a; 19 : hijk; 3 : bc; 7 : de; 8 : def; 9 : g;
Level 1:  : ; 1 : a; 7 : de;
Level 2:  : ; 1 : a;
Level 3:  : ;
Successfully deleted key 3
Successfully deleted key 7
skipList size:4

*****Skip List*****
Level 0:  : ; 1 : a; 19 : hijk; 8 : def; 9 : g;
Level 1:  : ; 1 : a;
Level 2:  : ; 1 : a;
```
