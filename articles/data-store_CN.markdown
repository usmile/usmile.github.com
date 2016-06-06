# DBDB: Dog Bed Database

作者：Taavi Burns

翻译：王世德

## 简介
DBDB (Dog Bed Database) 是一个用python实现的简单的键值数据库。可以将键值关联起来，并且可以将关联关系保存到磁盘上，用于后续的检索。

DBDB的目标在计算机宕机或者异常时，实现数据保存。并且避免将所有数据一次保存到内存中，这样就可以保存比RAM更多的数据。

## 回忆
还记得，第一次被一个bug搞得不知所措。当敲完我的一个BASIC程序运行时，诡异的像素出现在屏幕上，程序提前退出。回过头去看代码，后面的几行消失了。

我妈妈的一个朋友知道如何编程，打了个电话问了一下。和她交流几分钟后，找到了问题所在：程序太大了，侵占了视频内存。清理了一下被程序截断的屏幕，出现了在内存中存储的的苹果软件BASIC的正常行为。

从那刻开始，我对内粗分配都非常小心，学习了指针和使用malloc分配内存，学习了如何使用数据解决，学习非常仔细地改变内存。

几年后，当我阅读面向过程的语言Erlang的时候，了解到实际上没有必要在进程间拷贝数据来发送消息，因为所有的一切都是不可变的。然后发现在Clojure中不可变数据结构，并且真正进入到了内部。

在2013年，当阅读CouchDB时，了解到在变化时管理复杂数据的结构和机制。

了解到可以设计一个围绕不可变数据的系统。

然后打算写一本书的章节。



描述CouchDB的核心数据存储概念（以我理解的那样）会很有意思。

当开始尝试写一个原地改变结构的二叉树算法的时候，当越写越复杂的时候，令我非常失望。边界条件数量增加，树的一部分变化的时候可能会影响其它部分，让我非常伤心。没有任何办法解释所有的一切。

记住这些教训，使用递归算法更新不可变的二叉树，这样实现变得非常直接。

再一次学习到，不可变事物更容易推理。

故事开始了。

## 为什么很有意义
大部分项目需要某种类型的数据库。你不应该自己实现；有很多很多边界条件会折磨你，即使你仅仅将Json写入到磁盘。

* 文件系统超出空间会发生什么？
* 在保存时如果你的笔记本电池用完了，会发生什么？
* 当数据大小超出可用内存时又会发生什么？
  （不幸地，对于大部分现代桌面计算机和都会发生这种事情；对于移动设备和服务器web应用也会发生）

不管怎么样，如果想_理解_数据库如何处理所有的这些问题，自己写一个是一个很好的方式。

这里讨论的技术和概念可以应用到所有遇到故障之后，需要正确合理并且有可预期行为的问题上去。

话说故障...

## 描述故障
数据库通常以如何近似实现ACID特性为其特征：原子性、一致性、隔离性和持久性。

在DBDB中更新数据具有原子性和持久性，这两个特性在后面的章节会继续描述。DBDB不提供一致性保证，因为数据存储没有约束。隔离性也没有实现。

当然，应用程序可以自己实现实现的一致性保证，但合适的个隔离性需要事务管理器，这里就不尝试了，但可以在[CircleDB章节](http://aosabook.org/en/500L/an-archaeology-inspired-database.html)中学习。

还有一个系统维护的问题需要考虑，旧数据在这个实现中不再重新利用，所以重复更新（即使是相同的键）最终也会消耗磁盘空间。（后会看到为什么如此）PostgreSQL称其为回收利用（旧的空间可以重新使用），CouchDB称其为压缩（通过将数据重新写入到一个新的文件，自动将其移为就文件）。

DBDB可以通过增加压缩特性实现增强，但还是将其留给读者作为练习。

额外特性：你可以保证压缩树结构的平衡吗？这一特性提供性能保证。

## DBDB的架构
DBDB将关心的数据的物理层（数据在文件中如何保存）与数据的逻辑结构（这个例子中的二叉树）与键值存储的内容（键'a'与值'foo'的关联关系；公用API）分离开。

许多数据库将逻辑层与物理方面分离，这样可以选择不同的实现方式获得不同的性能，例如，DB2的SMS（在文件系统中的文件）与DMS（原始块存储）表空间，或者MySQL的[可选引擎实现](http://dev.mysql.com/doc/refman/5.7/en/storage-engines.html)。

## 探索设计
本书的所有章节都是描述一个程序是如何从头到尾构建完成的，但这种方式并不是我们工作时与代码交互方式。大部分情况下我们需要探索别人写的代码，并且需要给出如何修改或扩展来完成不同的事情。

在这一章中，假设DBDB是一个完成的项目，然后我们看下它如何工作，先来看下整个项目的结构。

### 组织单元
这里列出的内容与使用者的距离排序得到，第一个模块是用户需要最先了解的，最后一个是交互最少的一个。

1. ``tool.py``： 定义了一个命令行工具来研究数据库。
* ``interface.py``： 定义了一个类``DBDB``，其使用``二叉树``实现了python字典API，这是在python程序中使用DBDB的方式。
* ``logical.py``： 定义了逻辑层，键值存储的抽象接口。
    - ``LogicalBase``：提供了逻辑更新的API（像get，set和commit），具体的类实现自身的更新，并且管理存储锁定和去引用内部节点。
    - ``ValueRef``：是一个python对象用于引用数据库中保存的二进制blob，这种方式可以避免将所有数据一次全部存储到内存中。
* ``binary_tree.py``： 定义一个逻辑接口下具体的二叉树算法。
    - ``BinaryTree``： 给出二叉树具体的实现，并且有查找、插入、删除键值对的方法。``BinaryTree``代表不可变树；更新操作通过返回一个与旧树共享相同结构的新树实现。
    - ``BinaryNode``： 实现了二叉树的节点。
    - ``BinaryNodeRef``： 是一个具体的``ValueRef``，其实现了序列化和反序列化一个``BinaryNode``。
* ``physical.py``： 定义了一个物理层，``Storage``类提供了持久性，（大部分情况）append-only记录存储。

这些模块都是一个类仅完成一个功能。换句话说，每个类应该仅有个原因来改变。

### 读一个值
这里我们以最简单的例子开始：从数据库中读取一个值。

当尝试从``example.db``中关联的键``foo``获取值的时候会发生什么呢？
```bash
$ python -m dbdb.tool example.db get foo
```

这会运行模块``dbdb.tool``中的``main()``函数：
```python
# dbdb/tool.py
def main(argv):
    if not (4 <= len(argv) <= 5):
        usage()
        return BAD_ARGS
    dbname, verb, key, value = (argv[1:] + [None])[:4]
    if verb not in {'get', 'set', 'delete'}:
        usage()
        return BAD_VERB
    db = dbdb.connect(dbname)          # CONNECT
    try:
        if verb == 'get':
            sys.stdout.write(db[key])  # GET VALUE
        elif verb == 'set':
            db[key] = value
            db.commit()
        else:
            del db[key]
            db.commit()
    except KeyError:
        print("Key not found", file=sys.stderr)
        return BAD_KEY
    return OK
```

``connect()``函数打开数据文件（也可能是创建，但永远不会覆盖）返回``DBDB``的实例：
```python
# dbdb/__init__.py
def connect(dbname):
    try:
        f = open(dbname, 'r+b')
    except IOError:
        fd = os.open(dbname, os.O_RDWR | os.O_CREAT)
        f = os.fdopen(fd, 'r+b')
    return DBDB(f)
```

```python
# dbdb/interface.py
class DBDB(object):

    def __init__(self, f):
        self._storage = Storage(f)
        self._tree = BinaryTree(self._storage)
```

可以看到`DBDB`有一个`Storage`的实例引用，但其还共享了一个`self._tree`的引用。为什么呢？`self._tree`不能够完全管理访问自己的存储吗？

那个对象"拥有"一个资源通常是一个非常重要的设计考虑点，因为其会给出提示：对于什么类型的变化会不安全。我们急着这个问题，然后继续。

一旦有了一个DBDB实例，可以通过字典(``db[key]``)查找``key``的值，python解释器调用``DBDB.__getitem__()``。
```python
class DBDB(object):
# ...
    def __getitem__(self, key):
        self._assert_not_closed()
        return self._tree.get(key)

    def _assert_not_closed(self):
        if self._storage.closed:
            raise ValueError('Database closed.')
```

``__getitem__()``通过调用``_assert_not_closed_``来保证数据库仍然是打开的。啊哈！这里我们看到了至少一个原因为什么`DBDB`需要直接访问我们的`Storage`实例：它可以实现预置条件。（你是否赞成这个设计呢？你可以想到其它的方式来实现吗？）

DBDB通过``key``在内部``_tree``检索值通过调用``_tree.get()``，其提供了``LogicalBase``：
```python
# dbdb/logical.py
class LogicalBase(object):
# ...
    def get(self, key):
        if not self._storage.locked:
            self._refresh_tree_ref()
        return self._get(self._follow(self._tree_ref), key)
```

``get()``检查是否将存储锁定了，我们不是100%确定为什么这里需要锁，但可以猜测这里是为了允许写者来序列化访问数据。如果存储没有被锁定，会发生什么呢？

```python
# dbdb/logical.py
class LogicalBase(object):
# ...
def _refresh_tree_ref(self):
        self._tree_ref = self.node_ref_class(
            address=self._storage.get_root_address())
```

`_refresh_tree_ref`重置当前在磁盘上的树数据视图，允许执行完全更新的读请求。

当存储是锁定的时候，执行读操作，会发生什么呢？这意味着其它进程正在更改请求的数据，我们读到的数据可能不是最新的，这通常称之为“脏数据”，这种模式允许读者访问数据，但不用担心阻塞问题，但代价是可能读到过期的数据。

到这里，我们看下实际上如何检索数据：

```python
# dbdb/binary_tree.py
class BinaryTree(LogicalBase):
# ...
    def _get(self, node, key):
        while node is not None:
            if key < node.key:
                node = self._follow(node.left_ref)
            elif node.key < key:
                node = self._follow(node.right_ref)
            else:
                return self._follow(node.value_ref)
        raise KeyError
```

这是一个标准二叉树查找，后面引用其节点，阅读``BinaryTree``文档可以知道``Node``和``NodeRef``都是值对象：都是不可变的，其内容永远不会变化。

``Node``用一个关联的键值对来创建，这种关联也永远不会改变。整个``BinaryTree``的内容只有在根节点被替换的时候才是可见的，这意味着，当我们进行检索的时候，不用担心树的内容变化。

一旦关联的值被找到，会被``main()``函数写入到标准输出流中。

#### 插入和更新

