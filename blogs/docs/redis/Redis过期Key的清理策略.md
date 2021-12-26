[惰性清除](#惰性清除)  
[定期清除](#定期清除)  
[内存淘汰机制](#内存淘汰机制)  
[LRU算法](#LRU算法)

### 惰性清除
在访问key时，如果发现key已经过期，那么会将key删除，不给客户端返回任何东西。

### 定期清除
Redis默认会每秒进行10次过期扫描，过期扫描不会遍历过期字典中所有的 key，而是采用了一种简单的贪心策略。
- 从过期字典中随机20个key；
- 删除这20个key中已经过期的key；
- 如果过期的key比率超过1/4，那就重复上述步骤。

同时，为了保证过期扫描不会出现循环过度，导致线程卡死现象，算法还增加了扫描时间的上限，默认不会超过25ms。

### 内存淘汰机制
当执行写入命令时，如果发现内存不够，那么就会按照配置的淘汰策略清理内存。  
淘汰策略一般有6种，Redis4.0版本后又增加了2种，主要由分为三类：  

**第一类：不处理，等报错**  
noeviction：执行写入命令时发现内存不够直接返回错误信息。(Redis默认的配置)

**第二类：从所有结果集中的key中挑选，进行淘汰**  
allkeys-random：从所有的key中随机挑选key，进行淘汰；  
allkeys-lru：从所有的key中挑选最近使用时间距离现在最远的key，进行淘汰；  
allkeys-lfu：从所有的key中挑选使用频率最低的key，进行淘汰。(Redis4.0版本后新增的策略)  

**第三类：从设置了过期时间的key中挑选，进行淘汰**  
volatile-random：从设置了过期时间的结果集中随机挑选key删除；  
volatile-ttl：从设置了过期时间的结果集中挑选可存活时间最短的key开始删除(也就是从那些快要过期的key中先删除)；  
volatile-lru：从设置了过期时间的结果集中挑选上次使用时间距离现在最久的key开始删除；  
volatile-lfu：从过期时间的结果集中选择使用频率最低的key开始删除。(Redis4.0版本后新增的策略)  

### LRU算法
LRU(Least Recently Used)算法的设计原则是如果一个数据近期没有被访问到，那么之后一段时间都不会被访问到。  
所以当元素个数达到限制的值时，优先移除距离上次使用时间最久的元素。

那么如何实现呢？  
可以使用双向链表Node+HashMap<String, Node>来实现。  
每次访问元素后，将元素移动到链表头部；当元素满了时，将链表尾部的元素移除。  
HashMap主要用于根据key获得Node以及添加时判断节点是否已存在和删除时快速找到节点。  
```java
    //双向链表
    public static class ListNode {
        String key;//这里存储key。便于元素满时，删除尾节点时可以快速从HashMap删除键值对
        Integer value;
        ListNode pre = null;
        ListNode next = null;
        ListNode(String key, Integer value) {
            this.key = key;
            this.value = value;
        }
    }

    ListNode head;
    ListNode last;
    int limit=4;

    HashMap<String, ListNode> hashMap = new HashMap<String, ListNode>();

    public void add(String key, Integer val) {
        ListNode existNode = hashMap.get(key);
        if (existNode != null) {
            //从链表中删除这个元素
            ListNode pre = existNode.pre;
            ListNode next = existNode.next;
            if (pre!=null) {
               pre.next = next;
            }
            if (next!=null) {
               next.pre = pre;
            }
            //更新尾节点
            if (last==existNode) {
                last = existNode.pre;
            }
            //移动到最前面
            head.pre = existNode;
            existNode.next = head;
            head = existNode;
            //更新值
            existNode.value = val;
        } else {
            //达到限制，先删除尾节点
            if (hashMap.size() == limit) {
                ListNode deleteNode = last;
                hashMap.remove(deleteNode.key);
              //正是因为需要删除，所以才需要每个ListNode保存key
                last = deleteNode.pre;
                deleteNode.pre = null;
                last.next = null;
            }
            ListNode node = new ListNode(key,val);
            hashMap.put(key,node);
            if (head==null) {
                head = node;
                last = node;
            } else {
                //插入头结点
                node.next = head;
                head.pre = node;
                head = node;
            }
        }

    }

    public ListNode get(String key) {
        return hashMap.get(key);
    }

    public void remove(String key) {
        ListNode deleteNode = hashMap.get(key);
        ListNode preNode = deleteNode.pre;
        ListNode nextNode = deleteNode.next;
        if (preNode!=null) {
            preNode.next = nextNode;
        }
        if (nextNode!=null) {
            nextNode.pre = preNode;
        }
        if (head==deleteNode) {
            head = nextNode;
        }
        if (last == deleteNode) {
            last = preNode;
        }
        hashMap.remove(key);
    }
```

PS：使用单向链表能不能实现呢？
也可以。  
单向链表的节点虽然获取不到pre节点的信息，但是可以将下一个节点的key和value设置在当前节点上，然后把当前节点的next指针指向下下个节点，这样相当于把下一个节点删除了。