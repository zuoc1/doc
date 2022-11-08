# 1. STL基础

## 1.1 STL基本组成

| STL的组成  | 含义                                                         |
| ---------- | ------------------------------------------------------------ |
| 容器       | 一些封装数据结构的模板类，例如 vector 向量容器、list 列表容器等。 |
| 算法       | STL 提供了非常多（大约 100 个）的数据结构算法，它们都被设计成一个个的模板函数，这些算法在 std 命名空间中定义，其中大部分算法都包含在头文件 <algorithm> 中，少部分位于头文件 <numeric> 中。 |
| 迭代器     | 在C++ STL 中，对容器中数据的读和写，是通过迭代器完成的，扮演着容器和算法之间的胶合剂。 |
| 函数对象   | 如果一个类将 () 运算符重载为成员函数，这个类就称为函数对象类，这个类的对象就是函数对象（又称仿函数）。 |
| 适配器     | 可以使一个类的接口（模板的参数）适配成用户指定的形式，从而让原本不能在一起工作的两个类工作在一起。值得一提的是，容器、迭代器和函数都有适配器。 |
| 内存分配器 | 为容器类模板提供自定义的内存申请和释放功能，由于往往只有高级用户才有改变内存分配策略的需求，因此内存分配器对于一般用户来说，并不常用。 |

## 1.2 迭代器

当容器增加和删除元素，迭代器对应的内存发生变动，迭代器就会失效，需要重新生成迭代器。如vector增加大小导致重新分配内存、deque插入删除元素等。

迭代器的类型：输入迭代器、输出迭代器及以下三种

| 类型                                     | 支持的操作                                  | 说明                   |
| ---------------------------------------- | ------------------------------------------- | ---------------------- |
| 前向迭代器（forward iterator）           | ++p、p++、*p、==、!=、p1=p2                 | 只能比相等，不能比大小 |
| 双向迭代器（bidirectional iterator）     | 以上及--p、p--                              | 只能比相等，不能比大小 |
| 随机访问迭代器（random access iterator） | 以上及p+=i、p-=i、p[i]、<、>、<=、>=、p2-p1 | 可以用i随机访问        |

迭代器的定义方式：

| 迭代器定义方式 | 具体格式                                   | 说明                                                     |
| -------------- | ------------------------------------------ | -------------------------------------------------------- |
| 正向迭代器     | 容器类名::iterator 迭代器名;               |                                                          |
| 常量正向迭代器 | 容器类名::const_iterator 迭代器名;         |                                                          |
| 反向迭代器     | 容器类名::reverse_iterator 迭代器名;       | ++ 指的是迭代器向左移动一位，-- 指的是迭代器向右移动一位 |
| 常量反向迭代器 | 容器类名::const_reverse_iterator 迭代器名; |                                                          |

不同容器的迭代器

| 容器                               | 对应的迭代器类型         |
| ---------------------------------- | ------------------------ |
| array                              | 随机访问迭代器           |
| vector                             | 随机访问迭代器           |
| deque                              | 随机访问迭代器           |
| list                               | 双向迭代器               |
| set / multiset                     | 双向迭代器               |
| map / multimap                     | 双向迭代器               |
| forward_list                       | 前向迭代器               |
| unordered_map / unordered_multimap | 前向迭代器               |
| unordered_set / unordered_multiset | 前向迭代器               |
| stack                              | 容器适配器，不支持迭代器 |
| queue                              | 容器适配器，不支持迭代器 |
| priority_queue                     | 容器适配器，不支持迭代器 |

## 1.3 如何选择容器

总的来说，C++ STL 标准库（以 C++ 11 为准）提供了以下几种容器供我们选择：

1. 序列式容器：array、vector、deque、list 和 forward_list；
2. 关联式容器：map、multimap、set 和 multiset；
3. 无序关联式容器：unordered_map、unordered_multimap、unordered_set 和 unordered_multiset；
4. 容器适配器：stack、queue 和 priority_queue。

要想选择出适用于该特定场景的最佳容器，需要综合考虑多种实际因素，例如：

- 是否需要在容器的指定位置插入新元素？如果需要，则只能选择序列式容器，而关联式容器和哈希容器是不行的；
- 是否对容器中各元素的存储位置有要求？如果没有，则可以考虑使用哈希容器，反之就要避免使用哈希容器；
- 是否需要使用指定类型的迭代器？举个例子，如果必须是随机访问迭代器，则只能选择 array、vector、deque；如果必须是双向迭代器，则可以考虑 list 序列式容器以及所有的关联式容器；如果必须是前向迭代器，则可以考虑 forward_list 序列式容器以及所有的哈希容器；
- 当发生新元素的插入或删除操作时，是否要避免移动容器中的其它元素？如果是，则要避开 array、vector、deque，选择其它容器；
- 容器中查找元素的效率是否为关键的考虑因素？如果是，则应优先考虑哈希容器。

# 2. STL序列式容器

## 2.1 array

它是在 C++ 普通数组的基础上，添加了一些成员函数和全局函数。在使用上，它比普通数组更安全，且效率并没有因此变差。

### 2.1.1 声明初始化

```c++
// N必须是常量
template < class T, size_t N > class array;
```

```c++
#include <array>
using namespace std;

array<double, 10> values; // 声明
array<double, 10> values {}; // 全部初始化为0
array<double, 10> values {0.5,1.0,1.5,2.0}; // 剩下6个初始化为0
array<double, 10> values2 = values; // 类型和大小完全相同才能复制
```

### 2.1.2 随机访问迭代器

| 名称                 | 说明                                           | 例子/原型                                                    |
| -------------------- | ---------------------------------------------- | ------------------------------------------------------------ |
| begin() / cbegin()   | 返回指向容器中第一个元素的迭代器。             | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| end() / cend()       | 返回指向容器最后一个元素之后一个位置的迭代器。 | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| rbegin() / crbegin() | 返回指向最后一个元素的迭代器。                 | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| rend() / crend()     | 返回指向容器最后一个元素之后一个位置的迭代器。 | reverse_iterator rend() noexcept;<br/>const_reverse_iterator rend() const noexcept; |

### 2.1.3 容量

| 名称       | 说明                                                         | 例子/原型                                |
| ---------- | :----------------------------------------------------------- | ---------------------------------------- |
| size()     | 返回容器中当前元素的数量，其值始终等于初始化 array 类的第二个模板参数 N。 | constexpr size_type size() noexcept;     |
| max_size() | 返回容器可容纳元素的最大数量，其值始终等于初始化 array 类的第二个模板参数 N。 | constexpr size_type max_size() noexcept; |
| empty()    | 判断容器是否为空，和通过 size()==0 的判断条件功能相同，但其效率可能更快。 | constexpr bool empty() noexcept;         |

### 2.1.4 元素访问
| 名称       | 说明                                                         | 例子/原型                                                    |
| ---------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| operator[] | 返回容器中 n 位置处元素的引用，但不检查边界。                | reference operator[] (size_type n);<br/>const_reference operator[] (size_type n) const; |
| at(n)      | 返回容器中 n 位置处元素的引用，该函数自动检查 n 是否在有效的范围内，如果不是则抛出 out_of_range 异常。 | reference at ( size_type n );<br/>const_reference at ( size_type n ) const; |
| front()    | 返回容器中第一个元素的直接引用，该函数不适用于空的 array 容器。 | reference front();<br/>const_reference front() const;        |
| back()     | 返回容器中最后一个元素的直接应用，该函数同样不适用于空的 array 容器。 | reference back();<br/>const_reference back() const;          |
| data       | 返回一个指向容器首个元素的指针。利用该指针，可实现复制容器中所有元素等类似功能。 | value_type* data() noexcept;<br/>const value_type* data() const noexcept; |

### 2.1.5 修改
| 名称                | 说明                                                         | 例子/原型                          |
| ------------------- | :----------------------------------------------------------- | ---------------------------------- |
| fill(val)           | 将 val 这个值赋值给容器中的每个元素。                        | void fill (const value_type& val); |
| array1.swap(array2) | 交换 array1 和 array2 容器中的所有元素，但前提是它们具有相同的长度和类型。 |                                    |

### 2.1.6 非成员函数

| 名称                         | 说明                                                         | 例子/原型                                                    |
| ---------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| get (array)                  | 访问容器中指定的元素，并返回该元素的引用                     | array<int,3> myarray = {10, 20, 30};<br>tuple_element<0,decltype(myarray)>::type myelement = get<0>(myarray); |
| relational operators (array) | 关系运算法：==、<及其它                                      | a!=b  等价于 !(a==b)<br/>a>b   等价于   b<a<br/>a<=b 等价于 !(b<a)<br/>a>=b 等价于 !(a<b) |
| begin()                      | 既可以操作容器，也可以操作数组（返回第一个元素指针）。       | begin(myarray);等价于myarray.begin();                        |
| end()                        | 既可以操作容器，也可以操作数组（返回后一个元素之后一个位置的指针）。 | end(myarray);等价于myarray.end();                            |

### 2.1.7 非成员类特例化

| 名称                 | 说明                         | 例子/原型                                                    |
| -------------------- | ---------------------------- | ------------------------------------------------------------ |
| tuple_element<array> | 访问容器中指定位置元素的类型 | array<int,3> myarray = {10, 20, 30};<br/>tuple_element<0,decltype(myarray)>::type myelement = get<0>(myarray); |
| tuple_size<array>    | 返回容器size                 | array<int,3> myarray = {10, 20, 30};<br/>int size = tuple_size<decltype(myarray )>::value; |

## 2.2 vector

### 2.2.1 声明初始化

```c++
#include <vector>
using namespace std;

vector<double> values; // 空容器，没有分配内存
values.reserve(20); // 增加容器的容量

vector<int> values {2, 3, 5, 7, 11, 13, 17, 19};

vector<double> values(20); // 20个0
vector<double> values(20, 1.0); // 20个1.0

vector<int> value1{1,2,3,4,5};
vector<int> value2(value1); // 必须类型相同
vector<int> value2 = value1; // 必须类型相同

int array[]={1,2,3};
vector<int> values(array, array+2); // values 将保存{1,2}
vector<int> value2(begin(value1), begin(value1)+3); // value2保存{1,2,3}
```

### 2.2.2 随机访问迭代器

| 名称                 | 说明                                           | 例子/原型                                                    |
| -------------------- | ---------------------------------------------- | ------------------------------------------------------------ |
| begin() / cbegin()   | 返回指向容器中第一个元素的迭代器。             | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| end() / cend()       | 返回指向容器最后一个元素之后一个位置的迭代器。 | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| rbegin() / crbegin() | 返回指向最后一个元素的迭代器。                 | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| rend() / crend()     | 返回指向容器最后一个元素之后一个位置的迭代器。 | reverse_iterator rend() noexcept;<br/>const_reverse_iterator rend() const noexcept; |

### 2.2.3 容量

| 名称            | 说明                                                         | 例子/原型                                                    |
| --------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| size()          | 返回实际元素个数。                                           | constexpr size_type size() noexcept;                         |
| max_size()      | 返回元素个数的最大值。这通常是一个很大的值，一般是 232-1，所以我们很少会用到这个函数。 | constexpr size_type max_size() noexcept;                     |
| resize(n[,val]) | 改变实际元素的个数。n小于size，删除超出的元素(并销毁它们)；n大于size，在末尾插入元素（如果指定val用val初始化）；n大于capacity，自动重新分配存储空间。 | void resize (size_type n);<br/>void resize (size_type n, const value_type& val) |
| capacity()      | 返回当前容量。                                               | size_type capacity() const noexcept;                         |
| empty()         | 判断容器中是否有元素。                                       | constexpr bool empty() noexcept;                             |
| reserve(n)      | 增加容器的容量。n小于capacity，啥也不做。                    | void reserve (size_type n);                                  |
| shrink_to_fit() | 将内存减少到等于当前元素实际所使用的大小。vector<T>(x).swap(x);可达到同样效果。 | void shrink_to_fit();                                        |

### 2.2.4 元素访问
| 名称       | 说明                                                         | 例子/原型                                                    |
| ---------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| operator[] | 返回容器中 n 位置处元素的引用，但不检查边界。                | reference operator[] (size_type n);<br/>const_reference operator[] (size_type n) const; |
| at(n)      | 返回容器中 n 位置处元素的引用，该函数自动检查 n 是否在有效的范围内，如果不是则抛出 out_of_range 异常。 | reference at ( size_type n );<br/>const_reference at ( size_type n ) const; |
| front()    | 返回容器中第一个元素的直接引用。                             | reference front();<br/>const_reference front() const;        |
| back()     | 返回容器中最后一个元素的直接应用。                           | reference back();<br/>const_reference back() const;          |
| data()     | 返回指向容器中第一个元素的指针。                             | value_type* data() noexcept;<br/>const value_type* data() const noexcept; |

### 2.2.5 修改
| 名称           | 说明                                                         | 例子/原型                                                    |
| -------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| assign()       | 用新元素替换原有内容。                                       | (1)用迭代器：values.assign(other.begin(), other.end());<br>(2)填充3个1：values.assign(3, 1);<br/>(3)初始化列表：values.assign({1,2,3}); |
| push_back()    | 在序列的尾部添加一个元素。size减 1，capacity不变。           | void push_back (const value_type& val);<br/>void push_back (value_type&& val); |
| pop_back()     | 移出序列尾部的元素。                                         | void pop_back();                                             |
| insert()       | 在指定的位置插入一个或多个元素。                             | (1)插入1个3：values.insert(values.begin(), 3);<br/>(2)插入2个3：values.insert(values.begin(), 2, 3);<br/>(3)插入范围：values.insert(values.begin(), other.begin(), other.end());<br/>(4)初始化列表：values.insert(values.begin(), {1,2,3}); |
| erase()        | 移出一个元素或一段元素，返回删除元素下个位置的迭代器。size减小，capacity不变。 | (1)移出一个：values.erase(values.begin());<br/>(2)移出一段：values.erase(values.begin(), values.end()); |
| clear()        | 移出所有的元素，容器大小变为 0，容量不变。vector<T>().swap(x);也可清空容器。 | void clear() noexcept;                                       |
| swap()         | 交换两个容器的所有元素。                                     | values.swap(other);                                          |
| emplace()      | 在指定的位置直接生成一个元素。                               | template <class... Args><br/>iterator emplace (const_iterator position, Args&&... args); |
| emplace_back() | 在序列尾部生成一个元素。减少了构造和拷贝的次数，比push_back()更加高效。 | template <class... Args><br/>void emplace_back (Args&&... args); |

### 2.2.6 分配器

| 名称          | 说明                                 | 例子/原型                                      |
| ------------- | ------------------------------------ | ---------------------------------------------- |
| get_allocator | 返回与vector关联的分配器对象的副本。 | allocator_type get_allocator() const noexcept; |

### 2.2.7 非成员函数

| 名称                 | 说明                                                         | 例子/原型                                                    |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| relational operators | 关系运算法：==、<及其它                                      | a!=b  等价于 !(a==b)<br/>a>b   等价于   b<a<br/>a<=b 等价于 !(b<a)<br/>a>=b 等价于 !(a<b) |
| swap(x,y)            | 交换容器                                                     | swap(x,y); 等价于 x.swap(y);                                 |
| begin()              | 既可以操作容器，也可以操作数组（返回第一个元素指针）。       | begin(myarray);等价于myarray.begin();                        |
| end()                | 既可以操作容器，也可以操作数组（返回后一个元素之后一个位置的指针）。 | end(myarray);等价于myarray.end();                            |

### 2.2.8 模板特例化vector\<bool>

为了节省空间，vector\<bool> 底层在存储各个 bool 类型值时，每个 bool 值都只使用一个比特位（二进制位）来存储。可以选择使用 deque\<bool> 或者 bitset 来替代 vector\<bool>。

该专门化具有与非专门化vector相同的成员函数，但data、emplace和emplace_back不在此专门化中。

它补充了以下内容:

| 名称   | 说明                                              | 例子/原型                                                    |
| ------ | ------------------------------------------------- | ------------------------------------------------------------ |
| flip() | 翻转容器中的所有值:true变为false, false变为true。 | void flip() noexcept;                                        |
| swap() | 交换两个容器的所有元素。                          | values.swap(other); 和 values.swap(other.begin(), other.end()); |

## 2.3 deque

deque 是 double-ended queue 的缩写，又称双端队列容器。

和 vector 不同的是，deque 还擅长在序列头部添加或删除元素，所耗费的时间复杂度也为常数阶`O(1)`。并且更重要的一点是，**deque 容器中存储元素并不能保证所有元素都存储到连续的内存空间中**。

![](..\pic\deque.gif)

### 2.3.1 声明初始化

```c++
#include <deque>
using namespace std;

deque<double> values; // 空容器，没有分配内存
values.resize(20); // 增加容器的大小

deque<int> values {2, 3, 5, 7, 11, 13, 17, 19};

deque<double> values(20); // 20个0
deque<double> values(20, 1.0); // 20个1.0

deque<int> value1{1,2,3,4,5};
deque<int> value2(value1); // 必须类型相同
deque<int> value2 = value1; // 必须类型相同

int array[]={1,2,3};
deque<int> values(array, array+2); // values 将保存{1,2}
deque<int> value2(begin(value1), begin(value1)+3); // value2保存{1,2,3}
```

### 2.3.2 随机访问迭代器

| 名称                 | 说明                                           | 例子/原型                                                    |
| -------------------- | ---------------------------------------------- | ------------------------------------------------------------ |
| begin() / cbegin()   | 返回指向容器中第一个元素的迭代器。             | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| end() / cend()       | 返回指向容器最后一个元素之后一个位置的迭代器。 | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| rbegin() / crbegin() | 返回指向最后一个元素的迭代器。                 | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| rend() / crend()     | 返回指向容器最后一个元素之后一个位置的迭代器。 | reverse_iterator rend() noexcept;<br/>const_reverse_iterator rend() const noexcept; |

### 2.3.3 容量

| 名称            | 说明                                                         | 例子/原型                                                    |
| --------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| size()          | 返回实际元素个数。                                           | constexpr size_type size() noexcept;                         |
| max_size()      | 返回元素个数的最大值。这通常是一个很大的值，一般是 232-1，所以我们很少会用到这个函数。 | constexpr size_type max_size() noexcept;                     |
| resize(n[,val]) | 改变实际元素的个数。n小于size，删除超出的元素(并销毁它们)；n大于size，在末尾插入元素（如果指定val用val初始化）。 | void resize (size_type n);<br/>void resize (size_type n, const value_type& val) |
| empty()         | 判断容器中是否有元素。                                       | constexpr bool empty() noexcept;                             |
| shrink_to_fit() | 将内存减少到等于当前元素实际所使用的大小。deque<T>(x).swap(x);可达到同样效果。 | void shrink_to_fit();                                        |

### 2.3.4 元素访问

| 名称       | 说明                                                         | 例子/原型                                                    |
| ---------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| operator[] | 返回容器中 n 位置处元素的引用，但不检查边界。                | reference operator[] (size_type n);<br/>const_reference operator[] (size_type n) const; |
| at(n)      | 返回容器中 n 位置处元素的引用，该函数自动检查 n 是否在有效的范围内，如果不是则抛出 out_of_range 异常。 | reference at ( size_type n );<br/>const_reference at ( size_type n ) const; |
| front()    | 返回容器中第一个元素的直接引用。                             | reference front();<br/>const_reference front() const;        |
| back()     | 返回容器中最后一个元素的直接应用。                           | reference back();<br/>const_reference back() const;          |

### 2.3.5 修改

| 名称            | 说明                                                         | 例子/原型                                                    |
| --------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| assign()        | 用新元素替换原有内容。                                       | (1)用迭代器：values.assign(other.begin(), other.end());<br>(2)填充3个1：values.assign(3, 1);<br/>(3)初始化列表：values.assign({1,2,3}); |
| push_back()     | 在序列的尾部添加一个元素。size减 1，capacity不变。           | void push_back (const value_type& val);<br/>void push_back (value_type&& val); |
| push_front()    | 在序列的头部添加一个元素。                                   | void push_front (const value_type& val);<br/>void push_front (value_type&& val); |
| pop_back()      | 移出序列尾部的元素。                                         | void pop_back();                                             |
| pop_front()     | 移除容器头部的元素。                                         | void pop_front();                                            |
| insert()        | 在指定的位置插入一个或多个元素。                             | (1)插入1个3：values.insert(values.begin(), 3);<br/>(2)插入2个3：values.insert(values.begin(), 2, 3);<br/>(3)插入范围：values.insert(values.begin(), other.begin(), other.end());<br/>(4)初始化列表：values.insert(values.begin(), {1,2,3}); |
| erase()         | 移出一个元素或一段元素，返回删除元素下个位置的迭代器。size减小，capacity不变。 | (1)移出一个：values.erase(values.begin());<br/>(2)移出一段：values.erase(values.begin(), values.end()); |
| clear()         | 移出所有的元素，容器大小变为 0，容量不变。deque<T>().swap(x);也可清空容器。 | void clear() noexcept;                                       |
| swap()          | 交换两个容器的所有元素。                                     | values.swap(other);                                          |
| emplace()       | 在指定的位置直接生成一个元素。                               | template <class... Args><br/>iterator emplace (const_iterator position, Args&&... args); |
| emplace_back()  | 在容器头部生成一个元素。和 push_front() 的区别是，该函数直接在容器头部构造元素，省去了复制移动元素的过程。 | template <class... Args><br/>void emplace_back (Args&&... args); |
| emplace_front() | 在容器尾部生成一个元素。和 push_back() 的区别是，该函数直接在容器尾部构造元素，省去了复制移动元素的过程。 | template <class... Args><br/>void emplace_back (Args&&... args); |

### 2.3.6 分配器

| 名称          | 说明                                 | 例子/原型                                      |
| ------------- | ------------------------------------ | ---------------------------------------------- |
| get_allocator | 返回与vector关联的分配器对象的副本。 | allocator_type get_allocator() const noexcept; |

### 2.3.7 非成员函数

| 名称                 | 说明                                                         | 例子/原型                                                    |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| relational operators | 关系运算法：==、<及其它                                      | a!=b  等价于 !(a==b)<br/>a>b   等价于   b<a<br/>a<=b 等价于 !(b<a)<br/>a>=b 等价于 !(a<b) |
| swap(x,y)            | 交换容器                                                     | swap(x,y); 等价于 x.swap(y);                                 |
| begin()              | 既可以操作容器，也可以操作数组（返回第一个元素指针）。       | begin(myarray);等价于myarray.begin();                        |
| end()                | 既可以操作容器，也可以操作数组（返回后一个元素之后一个位置的指针）。 | end(myarray);等价于myarray.end();                            |

## 2.4 list

list 容器，又称双向链表容器，即该容器的底层是以双向链表的形式实现的。这意味着，list 容器中的元素可以分散存储在内存空间里，而不是必须存储在一整块连续的内存空间中。

### 2.4.1 声明初始化

```c++
#include <list>
using namespace std;

list<double> values; // 空容器

list<int> values {2, 3, 5, 7, 11, 13, 17, 19};

list<double> values(20); // 20个0
list<double> values(20, 1.0); // 20个1.0

list<int> value1{1,2,3,4,5};
list<int> value2(value1); // 必须类型相同
list<int> value2 = value1; // 必须类型相同

int array[]={1,2,3};
list<int> values(array, array+2); // values 将保存{1,2}
list<int> value2(begin(value1), begin(value1)+3); // value2保存{1,2,3}
```

### 2.4.2 双向迭代器

| 名称                 | 说明                                           | 例子/原型                                                    |
| -------------------- | ---------------------------------------------- | ------------------------------------------------------------ |
| begin() / cbegin()   | 返回指向容器中第一个元素的迭代器。             | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| end() / cend()       | 返回指向容器最后一个元素之后一个位置的迭代器。 | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| rbegin() / crbegin() | 返回指向最后一个元素的迭代器。                 | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| rend() / crend()     | 返回指向容器最后一个元素之后一个位置的迭代器。 | reverse_iterator rend() noexcept;<br/>const_reverse_iterator rend() const noexcept; |

### 2.4.3 容量

| 名称       | 说明                                                         | 例子/原型                                |
| ---------- | :----------------------------------------------------------- | ---------------------------------------- |
| size()     | 返回实际元素个数。                                           | constexpr size_type size() noexcept;     |
| max_size() | 返回元素个数的最大值。这通常是一个很大的值，一般是 232-1，所以我们很少会用到这个函数。 | constexpr size_type max_size() noexcept; |
| empty()    | 判断容器中是否有元素。                                       | constexpr bool empty() noexcept;         |

### 2.4.4 元素访问

| 名称    | 说明                               | 例子/原型                                             |
| ------- | :--------------------------------- | ----------------------------------------------------- |
| front() | 返回容器中第一个元素的直接引用。   | reference front();<br/>const_reference front() const; |
| back()  | 返回容器中最后一个元素的直接应用。 | reference back();<br/>const_reference back() const;   |

### 2.4.5 修改

| 名称             | 说明                                                         | 例子/原型                                                    |
| ---------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| assign()         | 用新元素替换原有内容。                                       | (1)用迭代器：values.assign(other.begin(), other.end());<br>(2)填充3个1：values.assign(3, 1);<br/>(3)初始化列表：values.assign({1,2,3}); |
| push_back()      | 在序列的尾部添加一个元素。size减 1，capacity不变。           | void push_back (const value_type& val);<br/>void push_back (value_type&& val); |
| push_front()     | 在序列的头部添加一个元素。                                   | void push_front (const value_type& val);<br/>void push_front (value_type&& val); |
| pop_back()       | 移出序列尾部的元素。                                         | void pop_back();                                             |
| pop_front()      | 移除容器头部的元素。                                         | void pop_front();                                            |
| insert()         | 在指定的位置插入一个或多个元素。                             | (1)插入1个3：values.insert(values.begin(), 3);<br/>(2)插入2个3：values.insert(values.begin(), 2, 3);<br/>(3)插入范围：values.insert(values.begin(), other.begin(), other.end());<br/>(4)初始化列表：values.insert(values.begin(), {1,2,3}); |
| erase()          | 移出一个元素或一段元素，返回删除元素下个位置的迭代器。size减小，capacity不变。 | (1)移出一个：values.erase(values.begin());<br/>(2)移出一段：values.erase(values.begin(), values.end()); |
| clear()          | 移出所有的元素，容器大小变为 0，容量不变。deque<T>().swap(x);也可清空容器。 | void clear() noexcept;                                       |
| swap()           | 交换两个容器的所有元素。                                     | values.swap(other);                                          |
| emplace()        | 在指定的位置直接生成一个元素。                               | template <class... Args><br/>iterator emplace (const_iterator position, Args&&... args); |
| emplace_back()   | 在容器头部生成一个元素。和 push_front() 的区别是，该函数直接在容器头部构造元素，省去了复制移动元素的过程。 | template <class... Args><br/>void emplace_back (Args&&... args); |
| emplace_front()  | 在容器尾部生成一个元素。和 push_back() 的区别是，该函数直接在容器尾部构造元素，省去了复制移动元素的过程。 | template <class... Args><br/>void emplace_back (Args&&... args); |
| resize(n[, val]) | 调整容器的大小。                                             | void resize (size_type n);<br/>void resize (size_type n, const value_type& val); |

### 2.4.6 操作

| 名称        | 说明                                                         | 例子/原型                                                    |
| ----------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| splice()    | 将一个 list 容器中的元素插入到另一个容器的指定位置。         | 见下                                                         |
| remove(val) | 删除容器中所有等于 val 的元素。                              | void remove (const value_type& val);                         |
| remove_if() | 删除容器中满足条件的元素。                                   | template \<class Predicate><br/>void remove_if (Predicate pred); |
| unique()    | 删除容器中相邻的重复元素，只保留一个。                       | void unique();<br/>template \<class BinaryPredicate><br/>void unique (BinaryPredicate binary_pred); |
| merge()     | 合并两个事先已排好序的 list 容器，并且合并之后的 list 容器依然是有序的。 | 见下                                                         |
| sort()      | 通过更改容器中元素的位置，将它们进行排序。                   | void sort();	<br/>template \<class Compare><br/>void sort (Compare comp); |
| reverse()   | 反转容器中元素的顺序。                                       | void reverse() noexcept;                                     |

splice：

```c++
list<int> mylist1 = {1, 2, 3, 4};
list<int> mylist2 = {10, 20, 30};
list<int>::iterator it = mylist1.begin() + 1; // points to 2

// 用法一：
mylist1.splice(it, mylist2); // mylist1: 1 10 20 30 2 3 4
                             // mylist2 (empty)
                             // "it" still points to 2 (the 5th element)
// 用法二：
mylist2.splice(mylist2.begin(), mylist1, it);
                                // mylist1: 1 10 20 30 3 4
                                // mylist2: 2
                                // "it" is now invalid.
it = mylist1.begin();
std::advance(it,3);            // "it" points now to 30
// 用法三：
mylist1.splice(mylist1.begin(), mylist1, it, mylist1.end());
                                // mylist1: 30 3 4 1 10 20
```

remove_if：

```c++
// list::remove_if
#include <iostream>
#include <list>

// a predicate implemented as a function:
bool single_digit (const int& value) { return (value<10); }

// a predicate implemented as a class:
struct is_odd {
  bool operator() (const int& value) { return (value%2)==1; }
};

int main ()
{
  int myints[]= {15,36,7,17,20,39,4,1};
  std::list<int> mylist (myints,myints+8);   // 15 36 7 17 20 39 4 1

  mylist.remove_if (single_digit);           // 15 36 17 20 39

  mylist.remove_if (is_odd());               // 36 20

  std::cout << "mylist contains:";
  for (std::list<int>::iterator it=mylist.begin(); it!=mylist.end(); ++it)
    std::cout << ' ' << *it;
  std::cout << '\n';

  return 0;
}
```

unique:

```c++
// list::unique
#include <iostream>
#include <cmath>
#include <list>

// a binary predicate implemented as a function:
bool same_integral_part (double first, double second)
{ return ( int(first)==int(second) ); }

// a binary predicate implemented as a class:
struct is_near {
  bool operator() (double first, double second)
  { return (fabs(first-second)<5.0); }
};

int main ()
{
  double mydoubles[]={ 12.15,  2.72, 73.0,  12.77,  3.14,
                       12.77, 73.35, 72.25, 15.3,  72.25 };
  std::list<double> mylist (mydoubles,mydoubles+10);
  
  mylist.sort();             //  2.72,  3.14, 12.15, 12.77, 12.77,
                             // 15.3,  72.25, 72.25, 73.0,  73.35

  mylist.unique();           //  2.72,  3.14, 12.15, 12.77
                             // 15.3,  72.25, 73.0,  73.35

  mylist.unique (same_integral_part);  //  2.72,  3.14, 12.15
                                       // 15.3,  72.25, 73.0

  mylist.unique (is_near());           //  2.72, 12.15, 72.25

  std::cout << "mylist contains:";
  for (std::list<double>::iterator it=mylist.begin(); it!=mylist.end(); ++it)
    std::cout << ' ' << *it;
  std::cout << '\n';

  return 0;
}
```

merge：

```c++
// list::merge
#include <iostream>
#include <list>

// compare only integral part:
bool mycomparison (double first, double second)
{ return ( int(first)<int(second) ); }

int main ()
{
  std::list<double> first, second;

  first.push_back (3.1);
  first.push_back (2.2);
  first.push_back (2.9);

  second.push_back (3.7);
  second.push_back (7.1);
  second.push_back (1.4);

  first.sort();
  second.sort();

  first.merge(second);

  // (second is now empty)

  second.push_back (2.1);

  first.merge(second,mycomparison);

  std::cout << "first contains:";
  for (std::list<double>::iterator it=first.begin(); it!=first.end(); ++it)
    std::cout << ' ' << *it;
  std::cout << '\n';

  return 0;
} // first contains: 1.4 2.2 2.9 2.1 3.1 3.7 7.1
```

### 2.4.7 分配器

| 名称          | 说明                                 | 例子/原型                                      |
| ------------- | ------------------------------------ | ---------------------------------------------- |
| get_allocator | 返回与vector关联的分配器对象的副本。 | allocator_type get_allocator() const noexcept; |

### 2.4.8 非成员函数

| 名称                 | 说明                                                         | 例子/原型                                                    |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| relational operators | 关系运算法：==、<及其它                                      | a!=b  等价于 !(a==b)<br/>a>b   等价于   b<a<br/>a<=b 等价于 !(b<a)<br/>a>=b 等价于 !(a<b) |
| swap(x,y)            | 交换容器                                                     | swap(x,y); 等价于 x.swap(y);                                 |
| begin()              | 既可以操作容器，也可以操作数组（返回第一个元素指针）。       | begin(myarray);等价于myarray.begin();                        |
| end()                | 既可以操作容器，也可以操作数组（返回后一个元素之后一个位置的指针）。 | end(myarray);等价于myarray.end();                            |

## 2.5 forward_list

forward_list 容器底层使用单链表，由于单链表没有双向链表那样灵活，因此相比 list 容器，forward_list 容器的功能受到了很多限制。

存储相同个数的同类型元素，单链表耗用的内存空间更少，空间利用率更高，并且对于实现某些操作单链表的执行效率也更高。

效率高是选用 forward_list 而弃用 list 容器最主要的原因，换句话说，只要是 list 容器和 forward_list 容器都能实现的操作，应优先选择 forward_list 容器。

### 2.5.1 声明初始化

```c++
#include <forward_list>
using namespace std;

forward_list<double> values; // 空容器

forward_list<int> values {2, 3, 5, 7, 11, 13, 17, 19};

forward_list<double> values(20); // 20个0
forward_list<double> values(20, 1.0); // 20个1.0

forward_list<int> value1{1,2,3,4,5};
forward_list<int> value2(value1); // 必须类型相同
forward_list<int> value2 = value1; // 必须类型相同

int array[]={1,2,3};
forward_list<int> values(array, array+2); // values 将保存{1,2}
forward_list<int> value2(begin(value1), begin(value1)+3); // value2保存{1,2,3}
```

### 2.5.2 前向代器

| 名称                             | 说明                                                     | 例子/原型                                                    |
| -------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| begin() / cbegin()               | 返回一个前向迭代器，其指向容器中第一个元素的位置。       | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| end() / cend()                   | 返回一个前向迭代器，其指向容器中最后一个元素之后的位置。 | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| before_begin() / cbefore_begin() | 返回一个前向迭代器，其指向容器中第一个元素之前的位置。   | iterator before_begin() noexcept;<br/>const_iterator before_begin() const noexcept; |

### 2.5.3 容量

| 名称       | 说明                                                         | 例子/原型                                |
| ---------- | :----------------------------------------------------------- | ---------------------------------------- |
| max_size() | 返回元素个数的最大值。这通常是一个很大的值，一般是 232-1，所以我们很少会用到这个函数。 | constexpr size_type max_size() noexcept; |
| empty()    | 判断容器中是否有元素。                                       | constexpr bool empty() noexcept;         |

### 2.5.4 元素访问

| 名称    | 说明                             | 例子/原型                                             |
| ------- | :------------------------------- | ----------------------------------------------------- |
| front() | 返回容器中第一个元素的直接引用。 | reference front();<br/>const_reference front() const; |

### 2.5.5 修改

| 名称             | 说明                                                         | 例子/原型                                                    |
| ---------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| assign()         | 用新元素替换原有内容。                                       | (1)用迭代器：values.assign(other.begin(), other.end());<br>(2)填充3个1：values.assign(3, 1);<br/>(3)初始化列表：values.assign({1,2,3}); |
| emplace_front()  | 在容器头部生成一个元素。该函数和 push_front() 的功能相同，但效率更高。 | template <class... Args><br/>void emplace_front (Args&&... args); |
| push_front()     | 在容器头部插入一个元素。                                     | void push_front (const value_type& val);<br/>void push_front (value_type&& val); |
| pop_front()      | 删除容器头部的一个元素。                                     | void pop_front();                                            |
| emplace_after()  | 在指定位置之后插入一个新元素，并返回一个指向新元素的迭代器。和 insert_after() 的功能相同，但效率更高。 | template <class... Args><br/>iterator emplace_after (const_iterator position, Args&&... args); |
| insert_after()   | 在指定位置之后插入一个新元素，并返回一个指向`插入区间最后一个元素`的迭代器。 | 见下                                                         |
| erase_after()    | 删除容器中某个指定位置或区域内的所有元素，返回删除元素后的一个元素迭代器。 | iterator erase_after (const_iterator position);<br/>iterator erase_after (const_iterator position, const_iterator last); |
| swap()           | 交换两个容器中的元素，必须保证这两个容器中存储的元素类型是相同的。 | values.swap(other);                                          |
| resize(n[, val]) | 调整容器的大小。                                             | void resize (size_type n);<br/>void resize (size_type n, const value_type& val); |
| clear()          | 删除容器存储的所有元素。                                     | void clear() noexcept;                                       |

emplace_after：

```c++
// forward_list::insert_after
#include <iostream>
#include <array>
#include <forward_list>

int main ()
{
  std::array<int,3> myarray = { 11, 22, 33 };
  std::forward_list<int> mylist;
  std::forward_list<int>::iterator it;

  // (1)插入一个
  it = mylist.insert_after ( mylist.before_begin(), 10 );          // 10
  // (2)插入多个                                                    //  ^  <- it
  it = mylist.insert_after ( it, 2, 20 );                          // 10 20 20
  // (3)插入区间                                                    //        ^
  it = mylist.insert_after ( it, myarray.begin(), myarray.end() ); // 10 20 20 11 22 33
                                                                   //                 ^
  it = mylist.begin();                                             //  ^
  // (4)插入初始化序列
  it = mylist.insert_after ( it, {1,2,3} );                        // 10 1 2 3 20 20 11 22 33
                                                                   //        ^

  std::cout << "mylist contains:";
  for (int& x: mylist) std::cout << ' ' << x;
  std::cout << '\n';
  return 0;
}
```

erase_after：

```c++
// erasing from forward_list
#include <iostream>
#include <forward_list>

int main ()
{
  std::forward_list<int> mylist = {10, 20, 30, 40, 50};

                                            // 10 20 30 40 50
  auto it = mylist.begin();                 // ^

  it = mylist.erase_after(it);              // 10 30 40 50
                                            //    ^
  it = mylist.erase_after(it,mylist.end()); // 10 30
                                            //       ^

  std::cout << "mylist contains:";
  for (int& x: mylist) std::cout << ' ' << x;
  std::cout << '\n';

  return 0;
}
```

### 2.5.6 操作

| 名称           | 说明                                                         | 例子/原型                                                    |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| splice_after() | 将某个 forward_list 容器中指定位置或区域内的元素插入到另一个容器的指定位置之后。 | 见下                                                         |
| remove(val)    | 删除容器中所有等于 val 的元素。                              | void remove (const value_type& val);                         |
| remove_if()    | 删除容器中满足条件的元素。                                   | template \<class Predicate><br/>void remove_if (Predicate pred); |
| unique()       | 删除容器中相邻的重复元素，只保留一个。                       | void unique();<br/>template \<class BinaryPredicate><br/>void unique (BinaryPredicate binary_pred); |
| merge()        | 合并两个事先已排好序的 list 容器，并且合并之后的 list 容器依然是有序的。 | 见下                                                         |
| sort()         | 通过更改容器中元素的位置，将它们进行排序。                   | void sort();	<br/>template \<class Compare><br/>void sort (Compare comp); |
| reverse()      | 反转容器中元素的顺序。                                       | void reverse() noexcept;                                     |

splice_after：

```c++
// forward_list::splice_after
#include <iostream>
#include <forward_list>

int main ()
{
  std::forward_list<int> first = { 1, 2, 3 };
  std::forward_list<int> second = { 10, 20, 30 };

  auto it = first.begin();  // points to the 1
  // 插入序列
  first.splice_after ( first.before_begin(), second );
                          // first: 10 20 30 1 2 3
                          // second: (empty)
                          // "it" still points to the 1 (now first's 4th element)
  // 插入区间，左开右开区间
  second.splice_after ( second.before_begin(), first, first.begin(), it);
                          // first: 10 1 2 3
                          // second: 20 30
  // 插入单个
  first.splice_after ( first.before_begin(), second, second.begin() );
                          // first: 30 10 1 2 3
                          // second: 20
                          // * notice that what is moved is AFTER the iterator

  std::cout << "first contains:";
  for (int& x: first) std::cout << " " << x;
  std::cout << std::endl;

  std::cout << "second contains:";
  for (int& x: second) std::cout << " " << x;
  std::cout << std::endl;

  return 0;
}
```

### 2.5.7 分配器

| 名称          | 说明                                 | 例子/原型                                      |
| ------------- | ------------------------------------ | ---------------------------------------------- |
| get_allocator | 返回与vector关联的分配器对象的副本。 | allocator_type get_allocator() const noexcept; |

### 2.5.8 非成员函数

| 名称                 | 说明                                                         | 例子/原型                                                    |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| relational operators | 关系运算法：==、<及其它                                      | a!=b  等价于 !(a==b)<br/>a>b   等价于   b<a<br/>a<=b 等价于 !(b<a)<br/>a>=b 等价于 !(a<b) |
| swap(x,y)            | 交换容器                                                     | swap(x,y); 等价于 x.swap(y);                                 |
| begin()              | 既可以操作容器，也可以操作数组（返回第一个元素指针）。       | begin(myarray);等价于myarray.begin();                        |
| end()                | 既可以操作容器，也可以操作数组（返回后一个元素之后一个位置的指针）。 | end(myarray);等价于myarray.end();                            |

### 2.5.9 获取size

forward_list 容器中是不提供 size() 函数的，但如果想要获取 forward_list 容器中存储元素的个数，可以使用头文件 \<iterator> 中的 distance() 函数。

```c++
#include <iostream>
#include <forward_list>
#include <iterator>

using namespace std;

int main()
{
    forward_list<int> my_words{1,2,3,4};
    int count = distance(begin(my_words), end(my_words));
    cout << count;
    return 0;
}
```

# 3. STL关联式容器详解

和序列式容器不同的是，关联式容器在存储元素时还会为每个元素在配备一个键，整体以键值对的方式存储到容器中。相比前者，关联式容器可以通过键值直接找到对应的元素，而无需遍历整个容器。另外，关联式容器在存储元素，默认会根据各元素键值的大小做升序排序。底层选用了「红黑树」这种数据结构来组织和存储各个键值对。

## 3.1 pair用法详解

### 3.1.1 声明初始化

```c++
#1) 默认构造函数，即创建空的 pair 对象
pair();
#2) 直接使用 2 个元素初始化成 pair 对象
pair (const first_type& a, const second_type& b);
#3) 拷贝（复制）构造函数，即借助另一个 pair 对象，创建新的 pair 对象
template<class U, class V> pair (const pair<U,V>& pr);
#4) 移动构造函数
template<class U, class V> pair (pair<U,V>&& pr);
#5) 使用右值引用参数，创建 pair 对象
template<class U, class V> pair (U&& a, V&& b);
```

例子：

```c++
// pair::pair example
#include <utility>      // std::pair, std::make_pair
#include <string>       // std::string
#include <iostream>     // std::cout

int main () {
  std::pair <std::string,double> product1;                     // default constructor
  std::pair <std::string,double> product2 ("tomatoes",2.30);   // value init
  std::pair <std::string,double> product3 (product2);          // copy constructor

  product1 = std::make_pair(std::string("lightbulbs"),0.99);   // using make_pair (move)

  product2.first = "shoes";                  // the type of first is string
  product2.second = 39.90;                   // the type of second is double

  std::cout << "The price of " << product1.first << " is $" << product1.second << '\n';
  std::cout << "The price of " << product2.first << " is $" << product2.second << '\n';
  std::cout << "The price of " << product3.first << " is $" << product3.second << '\n';
  return 0;
}
```

### 3.1.2 成员函数

| 名称   | 说明                                                         | 例子/原型  |
| ------ | ------------------------------------------------------------ | ---------- |
| swap() | 互换 2 个 pair 对象的键值对，其操作成功的前提是这 2 个 pair 对象的键和值的类型要相同。 | x.swap(y); |

### 3.1.3 非成员函数

| 名称                 | 说明                                             | 例子/原型                                                    |
| -------------------- | ------------------------------------------------ | ------------------------------------------------------------ |
| relational operators | 关系运算法：==、<及其它，先比较first再比较second | a!=b  等价于 !(a==b)<br/>a>b   等价于   b<a<br/>a<=b 等价于 !(b<a)<br/>a>=b 等价于 !(a<b) |
| swap(x,y)            | 交换容器                                         | swap(x,y); 等价于 x.swap(y);                                 |
| get (pair)           | 访问容器中指定的元素，并返回该元素的引用         | pair \<string,double> product ("tomatoes",2.30);<br/>int size = tuple_size<decltype(product )>::value; |

### 3.1.4 非成员类特例化

| 名称                 | 说明                         | 例子/原型                                                    |
| -------------------- | ---------------------------- | ------------------------------------------------------------ |
| tuple_element\<pair> | 访问容器中指定位置元素的类型 | pair \<string,double> product ("tomatoes",2.30);<br/>tuple_element<0,decltype(product )>::type myelement = get<0>(product ); |
| tuple_size\<pair>    | 返回2                        | pair \<string,double> product ("tomatoes",2.30);<br/>int size = tuple_size<decltype(product )>::value; |

## 3.2 map和multimap

作为关联式容器的一种，map 容器存储的都是 pair 对象，也就是用 pair 类模板创建的键值对。默认情况下，map 容器选用`std::less<T>`排序规则（其中 T 表示键的数据类型），其会根据键的大小对所有键值对做升序排序。当然，根据实际情况的需要，我们可以手动指定 map 容器的排序规则，既可以选用 STL标准库中提供的其它排序规则（比如`std::greater<T>`），也可以自定义排序规则。

另外需要注意的是，**使用 map 容器存储的各个键值对，键的值既不能重复也不能被修改。和 map 容器的区别在于，multimap 容器中可以同时存储多（≥2）个键相同的键值对。**

### 3.2.1 声明初始化

```c++
template < class Key,                                     // 指定键（key）的类型
           class T,                                       // 指定值（value）的类型
           class Compare = less<Key>,                     // 指定排序规则
           class Alloc = allocator<pair<const Key,T> >    // 指定分配器对象的类型
           > class map;
template < class Key,                                   // 指定键（key）的类型
           class T,                                     // 指定值（value）的类型
           class Compare = less<Key>,                   // 指定排序规则
           class Alloc = allocator<pair<const Key,T> >  // 指定分配器对象的类型
           > class multimap;
```

例子：multimap构造方法与map相同

```c++
#include <map>
using namespace std;

map<string, int> myMap; // 空容器

map<string, int> myMap{{"C语言教程", 10}, {"STL教程", 20}};
map<string, int> myMap{make_pair("C语言教程", 10), make_pair("STL教程", 20)};

// 创建一个会返回临时 map 对象的函数
map<string, int> disMap() {
    map<string, int> tempMap{{"C语言教程",10},{"STL教程",20}};
    return tempMap;
}
// 调用 map 类模板的移动构造函数创建 newMap 容器
map<string, int> newMap(disMap());

map<string, int, greater<string>> myMap{ {"C语言教程",10},{"STL教程",20} };
map<string, int> newMap(++myMap.begin(), myMap.end());
```

### 3.2.2 双向迭代器

| 名称                 | 说明                                           | 例子/原型                                                    |
| -------------------- | ---------------------------------------------- | ------------------------------------------------------------ |
| begin() / cbegin()   | 返回指向容器中第一个元素的迭代器。             | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| end() / cend()       | 返回指向容器最后一个元素之后一个位置的迭代器。 | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| rbegin() / crbegin() | 返回指向最后一个元素的迭代器。                 | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| rend() / crend()     | 返回指向容器最后一个元素之后一个位置的迭代器。 | reverse_iterator rend() noexcept;<br/>const_reverse_iterator rend() const noexcept; |

### 3.2.3 容量

| 名称       | 说明                                                         | 例子/原型                                |
| ---------- | :----------------------------------------------------------- | ---------------------------------------- |
| size()     | 返回实际元素个数。                                           | constexpr size_type size() noexcept;     |
| max_size() | 返回元素个数的最大值。这通常是一个很大的值，一般是 232-1，所以我们很少会用到这个函数。 | constexpr size_type max_size() noexcept; |
| empty()    | 判断容器中是否有元素。                                       | constexpr bool empty() noexcept;         |

### 3.2.4 元素访问

multimap 容器中指定的键可能对应多个键值对，**和 map 容器相比，multimap 未提供 at() 成员方法，也没有重载 [] 运算符。**

| 名称       | 说明                                                         | 例子/原型                                                    |
| ---------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| operator[] | map容器重载了 [] 运算符，只要知道 map 容器中某个键值对的键的值，就可以向获取数组中元素那样，通过键直接获取对应的值。 | mapped_type& operator[] (const key_type& k);<br/>mapped_type& operator[] (key_type&& k); |
| at(key)    | 找到 map 容器中 key 键对应的值，如果找不到，该函数会引发 out_of_range 异常。 | mapped_type& at (const key_type& k);<br/>const mapped_type& at (const key_type& k) const; |

### 3.2.5 修改

当实现“向 map 容器中添加新键值对元素”的操作时，insert() 成员方法的执行效率更高；而在实现“更新 map 容器指定键值对的值”的操作时，operator[ ] 的效率更高。

虽然 emplace_hint() 方法指定了插入键值对的位置，但 map 容器为了保持存储键值对的有序状态，可能会移动其位置。

| 名称           | 说明                                                         | 例子/原型                                                    |
| -------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| insert()       | 向 map 容器中插入键值对。                                    | 见下                                                         |
| erase()        | 删除 map 容器指定位置、指定键（key）值或者指定区域内的键值对。 | (1)	iterator  erase (const_iterator position);<br/>(2)	size_type erase (const key_type& k);<br/>(3)	iterator  erase (const_iterator first, const_iterator last); |
| swap()         | 交换 2 个 map 容器中存储的键值对，这意味着，操作的 2 个键值对的类型必须相同。 | x.swap(y);                                                   |
| clear()        | 清空 map 容器中所有的键值对，即使 map 容器的 size() 为 0。   | void clear() noexcept;                                       |
| emplace()      | 在当前 map 容器中的指定位置处构造新键值对。其效果和插入键值对一样，但效率更高。 | template <class... Args><br/>pair<iterator,bool> emplace (Args&&... args); |
| emplace_hint() | 在本质上和 emplace() 在 map 容器中构造新键值对的方式是一样的，不同之处在于，使用者必须为该方法提供一个指示键值对生成位置的迭代器，并作为该方法的第一个参数。 | template <class... Args><br/>iterator emplace_hint (const_iterator position, Args&&... args); |

insert：

```c++
// map::insert (C++98)
#include <iostream>
#include <map>

int main ()
{
  std::map<char,int> mymap;

  // (1) 单参数插入，返回pair<map<char,int>::iterator,bool>，迭代器指向新插入的值，另一个为结果
  mymap.insert ( std::pair<char,int>('a',100) );
  mymap.insert ( std::pair<char,int>('z',200) );

  std::pair<std::map<char,int>::iterator,bool> ret;
  ret = mymap.insert ( std::pair<char,int>('z',500) );
  if (ret.second==false) {
    std::cout << "element 'z' already existed";
    std::cout << " with a value of " << ret.first->second << '\n';
  }

  // (2) 指定位置插入，返回迭代器指向新插入的值
  std::map<char,int>::iterator it = mymap.begin();
  mymap.insert (it, std::pair<char,int>('b',300));  // max efficiency inserting
  mymap.insert (it, std::pair<char,int>('c',400));  // no max efficiency inserting

  // (3) 插入区间，返回void
  std::map<char,int> anothermap;
  anothermap.insert(mymap.begin(),mymap.find('c'));
  // (4) 插入初始化序列
  
  // showing contents:
  std::cout << "mymap contains:\n";
  for (it=mymap.begin(); it!=mymap.end(); ++it)
    std::cout << it->first << " => " << it->second << '\n';

  std::cout << "anothermap contains:\n";
  for (it=anothermap.begin(); it!=anothermap.end(); ++it)
    std::cout << it->first << " => " << it->second << '\n';

  return 0;
}
```

### 3.2.6 返回比较对象

| 名称         | 说明                                        | 例子/原型                                                 |
| ------------ | ------------------------------------------- | --------------------------------------------------------- |
| key_comp()   | 返回键比较对象，可以当函数使用比较map的键   | map<char,int>::key_compare mycomp = mymap.key_comp();     |
| value_comp() | 返回值比较对象，可以当函数使用比较map的pair | map<char,int>::value_compare mycomp = mymap.value_comp(); |

```c++
// map::key_comp
#include <iostream>
#include <map>

int main ()
{
  std::map<char,int> mymap;

  std::map<char,int>::key_compare mycomp = mymap.key_comp();

  mymap['a']=100;
  mymap['b']=200;
  mymap['c']=300;

  std::cout << "mymap contains:\n";

  char highest = mymap.rbegin()->first;     // key value of last element

  std::map<char,int>::iterator it = mymap.begin();
  do {
    std::cout << it->first << " => " << it->second << '\n';
  } while ( mycomp((*it++).first, highest) );

  std::cout << '\n';

  return 0;
}
```

```c++
// map::value_comp
#include <iostream>
#include <map>

int main ()
{
  std::map<char,int> mymap;

  mymap['x']=1001;
  mymap['y']=2002;
  mymap['z']=3003;

  std::cout << "mymap contains:\n";

  std::pair<char,int> highest = *mymap.rbegin();          // last element

  std::map<char,int>::iterator it = mymap.begin();
  do {
    std::cout << it->first << " => " << it->second << '\n';
  } while ( mymap.value_comp()(*it++, highest) );

  return 0;
}
```

### 3.2.7 操作

multimap主要通过以下函数操作键值对

| 名称             | 说明                                                         | 例子/原型                                                    |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| find(key)        | 在容器中查找首个键为 key 的键值对，如果成功找到，则返回指向该键值对的双向迭代器；反之，则返回和 end() 方法一样的迭代器。另外，如果 map 容器用 const 限定，则该方法返回的是 const 类型的双向迭代器。 | iterator find (const key_type& k);<br/>const_iterator find (const key_type& k) const; |
| count(key)       | 在当前容器中，查找键为 key 的键值对的个数并返回。注意，由于 map 容器中各键值对的键的值是唯一的，因此该函数的返回值最大为 1。 | size_type count (const key_type& k) const;                   |
| lower_bound(key) | 返回一个指向当前容器中第一个大于或等于 key 的键值对的双向迭代器。如果 map 容器用 const 限定，则该方法返回的是 const 类型的双向迭代器。 | iterator lower_bound (const key_type& k);<br/>const_iterator lower_bound (const key_type& k) const; |
| upper_bound(key) | 返回一个指向当前容器中第一个大于 key 的键值对的迭代器。如果 map 容器用 const 限定，则该方法返回的是 const 类型的双向迭代器。 | iterator upper_bound (const key_type& k);<br/>const_iterator upper_bound (const key_type& k) const; |
| equal_range(key) | 该方法返回一个 pair 对象（包含 2 个双向迭代器），其中 pair.first 和 lower_bound() 方法的返回值等价，pair.second 和 upper_bound() 方法的返回值等价。也就是说，该方法将返回一个范围，该范围中包含的键为 key 的键值对（map 容器键值对唯一，因此该范围最多包含一个键值对）。 | pair<const_iterator,const_iterator> equal_range (const key_type& k) const;<br/>pair<iterator,iterator> equal_range (const key_type& k); |

### 3.2.8 分配器

| 名称          | 说明                                 | 例子/原型                                      |
| ------------- | ------------------------------------ | ---------------------------------------------- |
| get_allocator | 返回与vector关联的分配器对象的副本。 | allocator_type get_allocator() const noexcept; |

### 3.2.9 非成员函数

| 名称    | 说明                                                         | 例子/原型                             |
| ------- | ------------------------------------------------------------ | ------------------------------------- |
| begin() | 既可以操作容器，也可以操作数组（返回第一个元素指针）。       | begin(myarray);等价于myarray.begin(); |
| end()   | 既可以操作容器，也可以操作数组（返回后一个元素之后一个位置的指针）。 | end(myarray);等价于myarray.end();     |

## 3.3 set和multiset

和 map、multimap 容器不同，使用 set 容器存储的各个键值对，要求键 key 和值 value 必须相等。multiset 容器和 set 容器唯一的差别在于，multiset 容器允许存储多个值相同的元素，而 set 容器中只能存储互不相同的元素。

另外，使用 set 容器存储的各个元素的值必须各不相同。更重要的是，从语法上讲 set 容器并没有强制对存储元素的类型做 const 修饰，即 set 容器中存储的元素的值是可以修改的。但是，C++ 标准为了防止用户修改容器中元素的值，对所有可能会实现此操作的行为做了限制，使得在正常情况下，用户是无法做到修改 set 容器中元素的值的。

对于初学者来说，切勿尝试直接修改 set 容器中已存储元素的值，这很有可能破坏 set 容器中元素的有序性，最正确的修改 set 容器中元素值的做法是：先删除该元素，然后再添加一个修改后的元素。

### 3.2.1 声明初始化

```c++
template < class T,                        // 键 key 和值 value 的类型
           class Compare = less<T>,        // 指定 set 容器内部的排序规则
           class Alloc = allocator<T>      // 指定分配器对象的类型
           > class set;
template < class T,                        // 存储元素的类型
           class Compare = less<T>,        // 指定容器内部的排序规则
           class Alloc = allocator<T> >    // 指定分配器对象的类型
           > class multiset;
```

例子：multiset构造方法与set相同

```c++
#include <set>
using namespace std;

set<string> myset; // 空容器

set<string> myset{"key1", "key2"};
set<string> copyset(myset);

set<string> retSet() {
    set<string> myset{"key1", "key2"};
    return myset;
}
set<string> copyset = retSet();

set<string, greater<string>> myse{"key1", "key2"};
set<string> copyset(++myset.begin(), myset.end());
```

### 3.2.2 双向迭代器

| 名称                 | 说明                                           | 例子/原型                                                    |
| -------------------- | ---------------------------------------------- | ------------------------------------------------------------ |
| begin() / cbegin()   | 返回指向容器中第一个元素的迭代器。             | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| end() / cend()       | 返回指向容器最后一个元素之后一个位置的迭代器。 | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| rbegin() / crbegin() | 返回指向最后一个元素的迭代器。                 | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| rend() / crend()     | 返回指向容器最后一个元素之后一个位置的迭代器。 | reverse_iterator rend() noexcept;<br/>const_reverse_iterator rend() const noexcept; |

### 3.2.3 容量

| 名称       | 说明                                                         | 例子/原型                                |
| ---------- | :----------------------------------------------------------- | ---------------------------------------- |
| size()     | 返回实际元素个数。                                           | constexpr size_type size() noexcept;     |
| max_size() | 返回元素个数的最大值。这通常是一个很大的值，一般是 232-1，所以我们很少会用到这个函数。 | constexpr size_type max_size() noexcept; |
| empty()    | 判断容器中是否有元素。                                       | constexpr bool empty() noexcept;         |

### 3.2.4 修改

虽然 emplace_hint() 方法指定了插入键值对的位置，但 map 容器为了保持存储键值对的有序状态，可能会移动其位置。

| 名称           | 说明                                                         | 例子/原型                                                    |
| -------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| insert()       | 向容器中插入键值对。                                         | 见下                                                         |
| erase()        | 删除 set 容器中存储的元素。                                  | (1)返回：指向容器中删除元素之后的第一个元素<br/>iterator  erase (const_iterator position);<br/>(2)返回成功删除的元素个数<br/>size_type erase (const value_type& val);<br/>(3)	iterator  erase (const_iterator first, const_iterator last); |
| swap()         | 交换 2 个 set 容器中存储的所有元素。这意味着，操作的 2 个 set 容器的类型必须相同。 | x.swap(y);                                                   |
| clear()        | 清空 set 容器中所有的元素，即令 set 容器的 size() 为 0。     | void clear() noexcept;                                       |
| emplace()      | 在当前 set 容器中的指定位置直接构造新元素。其效果和 insert() 一样，但效率更高。 | template <class... Args><br/>pair<iterator,bool> emplace (Args&&... args); |
| emplace_hint() | 在本质上和 emplace() 在 set 容器中构造新元素的方式是一样的，不同之处在于，使用者必须为该方法提供一个指示新元素生成位置的迭代器，并作为该方法的第一个参数。 | template <class... Args><br/>iterator emplace_hint (const_iterator position, Args&&... args); |

insert：

```c++
// set::insert (C++98)
#include <iostream>
#include <set>

int main ()
{
  std::set<int> myset;
  std::set<int>::iterator it;
  std::pair<std::set<int>::iterator,bool> ret;

  // (1) 设置初始化值，返回pair<iterator,bool>，迭代器指向新插入的值，另一个为结果
  for (int i=1; i<=5; ++i) myset.insert(i*10);    // set: 10 20 30 40 50

  ret = myset.insert(20);               // no new element inserted

  if (ret.second==false) it=ret.first;  // "it" now points to element 20

  // (2) 指定位置插入，返回迭代器指向新插入的值
  myset.insert (it,25);                 // max efficiency inserting
  myset.insert (it,24);                 // max efficiency inserting
  myset.insert (it,26);                 // no max efficiency inserting

  int myints[]= {5,10,15};              // 10 already in set, not inserted
  // (3) 插入区间，返回void
  myset.insert (myints,myints+3);
  // (4) 插入初始化序列

  std::cout << "myset contains:";
  for (it=myset.begin(); it!=myset.end(); ++it)
    std::cout << ' ' << *it;
  std::cout << '\n';

  return 0;
}
```

### 3.2.5 返回比较对象

| 名称         | 说明                                                         | 例子/原型                                             |
| ------------ | ------------------------------------------------------------ | ----------------------------------------------------- |
| key_comp()   | 返回键比较对象，可以当函数使用比较set的键                    | set\<int>::key_compare mycomp = myset.key_comp();     |
| value_comp() | 返回值比较对象，可以当函数使用比较set的值，键与值相同，所以同上 | set\<int>::value_compare mycomp = myset.value_comp(); |

```c++
// set::key_comp
#include <iostream>
#include <set>

int main ()
{
  std::set<int> myset;
  int highest;

  std::set<int>::key_compare mycomp = myset.key_comp();

  for (int i=0; i<=5; i++) myset.insert(i);

  std::cout << "myset contains:";

  highest=*myset.rbegin();
  std::set<int>::iterator it=myset.begin();
  do {
    std::cout << ' ' << *it;
  } while ( mycomp(*(++it),highest) );

  std::cout << '\n';

  return 0;
}
```

```c++
// set::value_comp
#include <iostream>
#include <set>

int main ()
{
  std::set<int> myset;

  std::set<int>::value_compare mycomp = myset.value_comp();

  for (int i=0; i<=5; i++) myset.insert(i);

  std::cout << "myset contains:";

  int highest=*myset.rbegin();
  std::set<int>::iterator it=myset.begin();
  do {
    std::cout << ' ' << *it;
  } while ( mycomp(*(++it),highest) );

  std::cout << '\n';

  return 0;
}
```

### 3.2.6 操作

multiset主要通过以下函数操作键值对

| 名称             | 说明                                                         | 例子/原型                                                    |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| find(key)        | 在容器中查找首个键为 key 的键值对，如果成功找到，则返回指向该键值对的双向迭代器；反之，则返回和 end() 方法一样的迭代器。另外，如果 map 容器用 const 限定，则该方法返回的是 const 类型的双向迭代器。 | iterator find (const key_type& k);<br/>const_iterator find (const key_type& k) const; |
| count(key)       | 在当前容器中，查找键为 key 的键值对的个数并返回。注意，由于 map 容器中各键值对的键的值是唯一的，因此该函数的返回值最大为 1。 | size_type count (const key_type& k) const;                   |
| lower_bound(key) | 返回一个指向当前容器中第一个大于或等于 key 的键值对的双向迭代器。如果 map 容器用 const 限定，则该方法返回的是 const 类型的双向迭代器。 | iterator lower_bound (const key_type& k);<br/>const_iterator lower_bound (const key_type& k) const; |
| upper_bound(key) | 返回一个指向当前容器中第一个大于 key 的键值对的迭代器。如果 map 容器用 const 限定，则该方法返回的是 const 类型的双向迭代器。 | iterator upper_bound (const key_type& k);<br/>const_iterator upper_bound (const key_type& k) const; |
| equal_range(key) | 该方法返回一个 pair 对象（包含 2 个双向迭代器），其中 pair.first 和 lower_bound() 方法的返回值等价，pair.second 和 upper_bound() 方法的返回值等价。也就是说，该方法将返回一个范围，该范围中包含的键为 key 的键值对（map 容器键值对唯一，因此该范围最多包含一个键值对）。 | pair<const_iterator,const_iterator> equal_range (const key_type& k) const;<br/>pair<iterator,iterator> equal_range (const key_type& k); |

### 3.2.7 分配器

| 名称          | 说明                                 | 例子/原型                                      |
| ------------- | ------------------------------------ | ---------------------------------------------- |
| get_allocator | 返回与vector关联的分配器对象的副本。 | allocator_type get_allocator() const noexcept; |

### 3.2.8 非成员函数

| 名称    | 说明                                                         | 例子/原型                             |
| ------- | ------------------------------------------------------------ | ------------------------------------- |
| begin() | 既可以操作容器，也可以操作数组（返回第一个元素指针）。       | begin(myarray);等价于myarray.begin(); |
| end()   | 既可以操作容器，也可以操作数组（返回后一个元素之后一个位置的指针）。 | end(myarray);等价于myarray.end();     |

## 3.4 自定义关联式容器的排序规则

### 3.4.1 使用函数对象自定义排序规则

```c++
#include <iostream>
#include <set>
#include <string>
using namespace std;
//定义函数对象类，struct和class非常类似，此处也可使用struct
class cmp {
public:
    //重载 () 运算符
    bool operator ()(const string &a,const string &b) {
        //按照字符串的长度，做升序排序(即存储的字符串从短到长)
        return  (a.length() < b.length());
    }
};

int main() {
    //创建 set 容器，并使用自定义的 cmp 排序规则
    set<string, cmp> myset{"http://c.biancheng.net/stl/", "http://c.biancheng.net/python/",
                           "http://c.biancheng.net/java/"};
    //输出容器中存储的元素
    for (auto iter = myset.begin(); iter != myset.end(); ++iter) {
            cout << *iter << endl;
    }
    return 0;
}
```

### 3.4.2 重载关系运算符实现自定义排序

C++ STL标准库适用于关联式容器的排序规则：

| 排序规则               | 功能                                                         |
| ---------------------- | ------------------------------------------------------------ |
| std::less\<T>          | 底层采用 < 运算符实现升序排序，各关联式容器默认采用的排序规则。 |
| std::greater\<T>       | 底层采用 > 运算符实现降序排序，同样适用于各个关联式容器。    |
| std::less_equal\<T>    | 底层采用 <= 运算符实现升序排序，多用于 multimap 和 multiset 容器。 |
| std::greater_equal\<T> | 底层采用 >= 运算符实现降序排序，多用于 multimap 和 multiset 容器。 |

值得一提的是，表 1 中的这些排序规则，其底层也是采用函数对象的方式实现的。以 std::less\<T> 为例，其底层实现为：

```c++
template <typename T>
struct less {
    //定义新的排序规则
    bool operator()(const T &_lhs, const T &_rhs) const {
        return _lhs < _rhs;
    }
}
```

在此基础上，当关联式容器中存储的数据类型为自定义的结构体变量或者类对象时，通过对现有排序规则中所用的关系运算符进行重载，也能实现自定义排序规则的目的。

> 注意，当关联式容器中存储的元素类型为结构体指针变量或者类的指针对象时，只能使用函数对象的方式自定义排序规则，此方法不再适用。

举个例子：

```c++
#include <iostream>
#include <set>      // set
#include <string>       // string
using namespace std;
//自定义类
class myString {
public:
    //定义构造函数，向 myset 容器中添加元素时会用到
    myString(string tempStr) :str(tempStr) {};
    //获取 str 私有对象，由于会被私有对象调用，因此该成员方法也必须为 const 类型
    string getStr() const;
private:
    string str;
};
string myString::getStr() const{
    return this->str;
}
//重载 < 运算符，参数必须都为 const 类型
bool operator <(const myString &stra, const myString & strb) {
    //以字符串的长度为标准比较大小
    return stra.getStr().length() < strb.getStr().length();
}
int main() {
    //创建空 set 容器，仍使用默认的 less<T> 排序规则
    std::set<myString>myset;
    //向 set 容器添加元素，这里会调用 myString 类的构造函数
    myset.emplace("http://c.biancheng.net/stl/");
    myset.emplace("http://c.biancheng.net/c/");
    myset.emplace("http://c.biancheng.net/python/");
    //
    for (auto iter = myset.begin(); iter != myset.end(); ++iter) {
        myString mystr = *iter;
        cout << mystr.getStr() << endl;
    }
    return 0;
}
// http://c.biancheng.net/c/
// http://c.biancheng.net/stl/
// http://c.biancheng.net/python/
```

另外，上面程序以全局函数的形式实现对 < 运算符的重载，还可以使用成员函数或者友元函数的形式实现。其中，当以成员函数的方式重载 < 运算符时，该成员函数必须声明为 const 类型，且参数也必须为 const 类型。至于参数的传值方式是采用按引用传递还是按值传递，都可以（建议采用按引用传递，效率更高）。

```c++
bool operator <(const myString & tempStr) const {
    //以字符串的长度为标准比较大小
    return this->str.length() < tempStr.str.length();
}

//--------------------------------------------------------

//类中友元函数的定义
friend bool operator <(const myString &a, const myString &b);
//类外部友元函数的具体实现
bool operator <(const myString &stra, const myString &strb) {
    //以字符串的长度为标准比较大小
    return stra.str.length() < strb.str.length();
}
```

## 3.5 修改关联式容器中键值对的键

首先可以明确的是，map 和 multimap 容器只能采用“先删除，再添加”的方式修改某个键值对的键。原因很简单，C++ STL 标准中明确规定，map 和 multimap 容器用于存储类型为 pair<const K, V> 的键值对。显然，只要目标键值对存储在当前容器中，键的值就无法被修改。

和 map、multimap 不同，C++ STL 标准中并没有用 const 限定 set 和 multiset 容器中存储元素的类型。换句话说，对于 set<T> 或者 multiset<T> 类型的容器，其存储元素的类型是 T 而不是 const T。

例子：

```c++
// 定义学生类
class student {
public:
    student(string name, int id) : name(name), id(id) {
    }
    const int& getid() const {
        return id;
    }
    void setname(const string name){
        this->name = name;
    }
    string getname() const{
        return  name;
    }
private:
    string name;
    int id;
};

// 定义排序规则
class cmp {
public:
    bool operator ()(const student &stua, const student &stub) {
        //按照字符串的长度，做升序排序(即存储的字符串从短到长)
        return  stua.getid() < stub.getid();
    }
};

// 定义set容器
set<student, cmp> myset{ {"zhangsan",10},{"lisi",20},{"wangwu",15} };
set<student>::iterator iter = mymap.begin();
// 无法编译通过，因为*iter返回const T&
(*iter).setname("xiaoming");
// 正常修改name，但id作为真正的键无法修改
const_cast<student&>(*iter).setname("xiaoming");
```

注意，set 容器中每个元素也可以看做是键和值相等的键值对，但对于这里的 myset 容器来说，其实每个 student 对象的 id 才是真正的键，其它信息（name 和 age）只不过是和 id 绑定在一起而已。因此，在不破坏 myset 容器中元素的有序性的前提下（即不修改每个学生的 id），学生的其它信息是应该允许修改的，但有一个前提，即 myset 容器中存储的各个 student 对象不能被 const 修饰（这也是 set 容器中的元素类型不能被 const 修饰的原因）。

# 4. STL无序关联式容器

无序关联式容器，又称哈希容器。和关联式容器一样，此类容器存储的也是键值对元素；不同之处在于，关联式容器默认情况下会对存储的元素做升序排序，而无序关联式容器不会。实际场景中如果涉及大量遍历容器的操作，建议首选关联式容器；反之，如果更多的操作是通过键获取对应的值，则应首选无序容器。

1. 无序容器内部存储的键值对是无序的，各键值对的存储位置取决于该键值对中的键，
2. 和关联式容器相比，无序容器擅长通过指定键查找对应的值（平均时间复杂度为 O(1)）；但对于使用迭代器遍历容器中存储的元素，无序容器的执行效率则不如关联式容器。

## 4.1 unordered_map和unordered_multimap

具体来讲，unordered_map 容器和 map 容器一样，以键值对（pair类型）的形式存储数据，存储的各个键值对的键互不相同且不允许被修改。但由于 unordered_map 容器底层采用的是哈希表存储结构，该结构本身不具有对数据的排序功能，所以此容器内部不会自行对存储的键值对进行排序。

unordered_multimap 容器可以存储多个键相等的键值对，而 unordered_map 容器不行。

### 4.1.1 声明初始化

自定义类型hash\<Key>和equal_to\<Key>都将失效，需要自定义。

```c++
template < class Key,                        //键值对中键的类型
           class T,                          //键值对中值的类型
           class Hash = hash<Key>,           //容器内部存储键值对所用的哈希函数
           class Pred = equal_to<Key>,       //判断各个键值对键相同的规则
           class Alloc = allocator< pair<const Key,T> >  // 指定分配器对象的类型
           > class unordered_map;
template < class Key,      //键（key）的类型
           class T,        //值（value）的类型
           class Hash = hash<Key>,  //底层存储键值对时采用的哈希函数
           class Pred = equal_to<Key>,  //判断各个键值对的键相等的规则
           class Alloc = allocator< pair<const Key,T> > // 指定分配器对象的类型
           > class unordered_multimap;
```

例子：

```c++
#include <unordered_map>
using namespace std;

unordered_map<string, int> myMap; // 空容器

unordered_map<string, int> myMap{{"C语言教程", 10}, {"STL教程", 20}};
unordered_map<string, int> myMap{make_pair("C语言教程", 10), make_pair("STL教程", 20)};

unordered_map<string, int> newMap(myMap);

unordered_map<string, int> newMap(++myMap.begin(), myMap.end());
```

### 4.1.2 正向迭代器

| 名称               | 说明                                           | 例子/原型                                                    |
| ------------------ | ---------------------------------------------- | ------------------------------------------------------------ |
| begin() / cbegin() | 返回指向容器中第一个元素的迭代器。             | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| end() / cend()     | 返回指向容器最后一个元素之后一个位置的迭代器。 | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |

### 4.1.3 容量

| 名称       | 说明                                                         | 例子/原型                                |
| ---------- | :----------------------------------------------------------- | ---------------------------------------- |
| size()     | 返回实际元素个数。                                           | constexpr size_type size() noexcept;     |
| max_size() | 返回元素个数的最大值。这通常是一个很大的值，一般是 232-1，所以我们很少会用到这个函数。 | constexpr size_type max_size() noexcept; |
| empty()    | 判断容器中是否有元素。                                       | constexpr bool empty() noexcept;         |

### 4.1.4 元素访问

**和 unordered_map 容器相比，unordered_multimap 未提供 at() 成员方法，也没有重载 [] 运算符。**

| 名称       | 说明                                                         | 例子/原型                                                    |
| ---------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| operator[] | map容器重载了 [] 运算符，只要知道 map 容器中某个键值对的键的值，就可以向获取数组中元素那样，通过键直接获取对应的值。 | mapped_type& operator[] (const key_type& k);<br/>mapped_type& operator[] (key_type&& k); |
| at(key)    | 找到 map 容器中 key 键对应的值，如果找不到，该函数会引发 out_of_range 异常。 | mapped_type& at (const key_type& k);<br/>const mapped_type& at (const key_type& k) const; |

### 4.1.5 元素查找

unordered_multimap主要使用如下方法获取元素。

| 名称             | 说明                                                         | 例子/原型                                                    |
| ---------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| find(key)        | 查找以 key 为键的键值对，如果找到，则返回一个指向该键值对的正向迭代器；反之，则返回一个指向容器中最后一个键值对之后位置的迭代器（如果 end() 方法返回的迭代器）。 | iterator find ( const key_type& k );<br/>const_iterator find ( const key_type& k ) const; |
| count(key)       | 在容器中查找以 key 键的键值对的个数。                        | size_type count ( const key_type& k ) const;                 |
| equal_range(key) | 返回一个 pair 对象，其包含 2 个迭代器，用于表明当前容器中键为 key 的键值对所在的范围。 | pair<iterator,iterator> equal_range(const key_type& k);      |

### 4.1.6 修改

可以这样理解，emplace_hint() 方法中传入的迭代器，仅是给 unordered_map 容器提供一个建议，并不一定会被容器采纳。

| 名称           | 说明                                                         | 例子/原型                                                    |
| -------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| insert()       | 向容器中添加新键值对。                                       | 见下                                                         |
| erase()        | 删除指定键值对。                                             | (1)	iterator  erase (const_iterator position);<br/>(2)	size_type erase (const key_type& k);<br/>(3)	iterator  erase (const_iterator first, const_iterator last); |
| swap()         | 交换 2 个 map 容器中存储的键值对，这意味着，操作的 2 个键值对的类型必须相同。 | x.swap(y);                                                   |
| clear()        | 清空容器，即删除容器中存储的所有键值对。                     | void clear() noexcept;                                       |
| emplace()      | 向容器中添加新键值对，效率比 insert() 方法高。               | template <class... Args><br/>pair<iterator,bool> emplace (Args&&... args); |
| emplace_hint() | 指定位置向容器中添加新键值对，效率比 insert() 方法高。       | template <class... Args><br/>iterator emplace_hint (const_iterator position, Args&&... args); |

insert：

```c++
// unordered_map::insert
#include <iostream>
#include <string>
#include <unordered_map>

int main ()
{
  std::unordered_map<std::string,double>
              myrecipe,
              mypantry = {{"milk",2.0},{"flour",1.5}};

  std::pair<std::string,double> myshopping ("baking powder",0.3);

  // (1) 插入一个值，返回pair<iterator,bool>，迭代器指向新插入的值，另一个为结果
  myrecipe.insert (myshopping);                        // copy insertion
  myrecipe.insert (std::make_pair<std::string,double>("eggs",6.0)); // move insertion
  // (2) 指定位置插入，返回迭代器指向新插入的值，没实现
  // (3) 插入区间，返回void
  myrecipe.insert (mypantry.begin(), mypantry.end());  // range insertion
  // (4) 插入初始化序列
  myrecipe.insert ( {{"sugar",0.8},{"salt",0.1}} );    // initializer list insertion

  std::cout << "myrecipe contains:" << std::endl;
  for (auto& x: myrecipe)
    std::cout << x.first << ": " << x.second << std::endl;

  std::cout << std::endl;
  return 0;
}
```

### 4.1.7 桶函数

| 名称               | 说明                                                         | 例子/原型                                     |
| ------------------ | ------------------------------------------------------------ | --------------------------------------------- |
| bucket_count()     | 返回当前容器底层存储键值对时，使用桶（一个线性链表代表一个桶）的数量。 | size_type bucket_count() const noexcept;      |
| max_bucket_count() | 返回当前系统中，unordered_map 容器底层最多可以使用多少桶。   | size_type max_bucket_count() const noexcept;  |
| bucket_size(n)     | 返回第 n 个桶中存储键值对的数量。                            | size_type bucket_size ( size_type n ) const;  |
| bucket(key)        | 返回以 key 为键的键值对所在桶的编号。                        | size_type bucket ( const key_type& k ) const; |

### 4.1.8 哈希策略

默认情况下，无序容器的最大负载因子为 1.0。如果操作无序容器过程中，使得最大复杂因子超过了默认值，则容器会自动增加桶数，并重新进行哈希，以此来减小负载因子的值。需要注意的是，此过程会导致容器迭代器失效，但指向单个键值对的引用或者指针仍然有效。

| 名称              | 说明                                                         | 例子/原型                                                    |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| load_factor()     | 返回 unordered_map 容器中当前的负载因子。负载因子，指的是的当前容器中存储键值对的数量（size()）和使用桶数（bucket_count()）的比值，即 load_factor() = size() / bucket_count()。 | float load_factor() const noexcept;                          |
| max_load_factor() | 返回或者设置当前 unordered_map 容器的负载因子。              | get (1)	float max_load_factor() const noexcept;<br/>set (2)	void max_load_factor ( float z ); |
| rehash(n)         | 将当前容器底层使用桶的数量设置为 n。                         | void rehash( size_type n );                                  |
| reserve()         | 将存储桶的数量（也就是 bucket_count() 方法的返回值）设置为至少容纳count个元（不超过最大负载因子）所需的数量，并重新整理容器。 | void reserve ( size_type n );                                |

### 4.1.9 观察者

| 名称            | 说明                                | 例子/原型                                      |
| --------------- | ----------------------------------- | ---------------------------------------------- |
| hash_function() | 返回当前容器使用的哈希函数对象。    | hasher hash_function() const;                  |
| key_eq()        | 返回当前容器使用的key相等函数对象。 | key_equal key_eq() const;                      |
| get_allocator   | 返回用于构造容器的分配器对象。      | allocator_type get_allocator() const noexcept; |

### 4.1.10 非成员函数

| 名称                      | 说明                                                         | 例子/原型                             |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------- |
| operators (unordered_map) | 关系运算符：==、!=                                           |                                       |
| swap (unordered_map)      | 交换容器                                                     | swap(x,y); 等价于 x.swap(y);          |
| begin()                   | 既可以操作容器，也可以操作数组（返回第一个元素指针）。       | begin(myarray);等价于myarray.begin(); |
| end()                     | 既可以操作容器，也可以操作数组（返回后一个元素之后一个位置的指针）。 | end(myarray);等价于myarray.end();     |

## 4.2 unordered_set和unordered_set

总的来说，unordered_set 容器具有以下几个特性：

1. 不再以键值对的形式存储数据，而是直接存储数据的值；
2. 容器内部存储的各个元素的值都互不相等，且不能被修改。
3. 不会对内部存储的数据进行排序；

和 unordered_set 容器不同的是，unordered_multiset 容器可以同时存储多个值相同的元素，且这些元素会存储到哈希表中同一个桶（本质就是链表）上。

### 4.2.1 声明初始化

自定义类型hash\<Key>和equal_to\<Key>都将失效，需要自定义。

```c++
template < class Key,            //容器中存储元素的类型
           class Hash = hash<Key>,    //确定元素存储位置所用的哈希函数
           class Pred = equal_to<Key>,   //判断各个元素是否相等所用的函数
           class Alloc = allocator<Key>   //指定分配器对象的类型
           > class unordered_set;
template < class Key,            //容器中存储元素的类型
           class Hash = hash<Key>,    //确定元素存储位置所用的哈希函数
           class Pred = equal_to<Key>,   //判断各个元素是否相等所用的函数
           class Alloc = allocator<Key>   //指定分配器对象的类型
           > class unordered_multiset;
```

例子：

```c++
#include <unordered_set>
using namespace std;

unordered_set<string> mySet; // 空容器

unordered_set<string> mySet{"C语言教程", "STL教程"};
unordered_set<string> newSet(mySet);

unordered_set<string> newSet(++mySet.begin(), mySet.end());
```

### 4.2.2 正向迭代器

由于 unordered_set 容器内部存储的元素值不能被修改，因此无论使用那个迭代器方法获得的迭代器，都不能用于修改容器中元素的值。

| 名称               | 说明                                           | 例子/原型                                                    |
| ------------------ | ---------------------------------------------- | ------------------------------------------------------------ |
| begin() / cbegin() | 返回指向容器中第一个元素的迭代器。             | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |
| end() / cend()     | 返回指向容器最后一个元素之后一个位置的迭代器。 | iterator begin() noexcept;<br/>const_iterator begin() const noexcept; |

### 4.2.3 容量

| 名称       | 说明                                                         | 例子/原型                                |
| ---------- | :----------------------------------------------------------- | ---------------------------------------- |
| size()     | 返回实际元素个数。                                           | constexpr size_type size() noexcept;     |
| max_size() | 返回元素个数的最大值。这通常是一个很大的值，一般是 232-1，所以我们很少会用到这个函数。 | constexpr size_type max_size() noexcept; |
| empty()    | 判断容器中是否有元素。                                       | constexpr bool empty() noexcept;         |

### 4.2.4 元素查找

unordered_multiset主要使用如下方法获取元素。

| 名称             | 说明                                                         | 例子/原型                                                    |
| ---------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| find(key)        | 查找以 key 为键的键值对，如果找到，则返回一个指向该键值对的正向迭代器；反之，则返回一个指向容器中最后一个键值对之后位置的迭代器（如果 end() 方法返回的迭代器）。 | iterator find ( const key_type& k );<br/>const_iterator find ( const key_type& k ) const; |
| count(key)       | 在容器中查找以 key 键的键值对的个数。                        | size_type count ( const key_type& k ) const;                 |
| equal_range(key) | 返回一个 pair 对象，其包含 2 个迭代器，用于表明当前容器中键为 key 的键值对所在的范围。 | pair<iterator,iterator> equal_range(const key_type& k);      |

### 4.2.5 修改

可以这样理解，emplace_hint() 方法中传入的迭代器，仅是给容器提供一个建议，并不一定会被容器采纳。

| 名称           | 说明                                                         | 例子/原型                                                    |
| -------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| insert()       | 向容器中添加新键值对。                                       | 见下                                                         |
| erase()        | 删除指定键值对。                                             | (1)	iterator  erase (const_iterator position);<br/>(2)	size_type erase (const key_type& k);<br/>(3)	iterator  erase (const_iterator first, const_iterator last); |
| swap()         | 交换 2 个 map 容器中存储的键值对，这意味着，操作的 2 个键值对的类型必须相同。 | x.swap(y);                                                   |
| clear()        | 清空容器，即删除容器中存储的所有键值对。                     | void clear() noexcept;                                       |
| emplace()      | 向容器中添加新键值对，效率比 insert() 方法高。               | template <class... Args><br/>pair<iterator,bool> emplace (Args&&... args); |
| emplace_hint() | 指定位置向容器中添加新键值对，效率比 insert() 方法高。       | template <class... Args><br/>iterator emplace_hint (const_iterator position, Args&&... args); |

insert：

```c++
// unordered_set::insert
#include <iostream>
#include <string>
#include <array>
#include <unordered_set>

int main ()
{
  std::unordered_set<std::string> myset = {"yellow","green","blue"};
  std::array<std::string,2> myarray = {"black","white"};
  std::string mystring = "red";

  // (1) 插入一个值，返回pair<iterator,bool>，迭代器指向新插入的值，另一个为结果
  myset.insert (mystring);                        // copy insertion
  myset.insert (mystring+"dish");                 // move insertion
  // (2) 指定位置插入，返回迭代器指向新插入的值，没实现
  // (3) 插入区间，返回void
  myset.insert (myarray.begin(), myarray.end());  // range insertion
  // (4) 插入初始化序列
  myset.insert ( {"purple","orange"} );           // initializer list insertion

  std::cout << "myset contains:";
  for (const std::string& x: myset) std::cout << " " << x;
  std::cout <<  std::endl;

  return 0;
}
```

### 4.2.6 桶函数

| 名称               | 说明                                                         | 例子/原型                                     |
| ------------------ | ------------------------------------------------------------ | --------------------------------------------- |
| bucket_count()     | 返回当前容器底层存储键值对时，使用桶（一个线性链表代表一个桶）的数量。 | size_type bucket_count() const noexcept;      |
| max_bucket_count() | 返回当前系统中，容器底层最多可以使用多少桶。                 | size_type max_bucket_count() const noexcept;  |
| bucket_size(n)     | 返回第 n 个桶中存储键值对的数量。                            | size_type bucket_size ( size_type n ) const;  |
| bucket(key)        | 返回以 key 为键的键值对所在桶的编号。                        | size_type bucket ( const key_type& k ) const; |

### 4.2.7 哈希策略

默认情况下，无序容器的最大负载因子为 1.0。如果操作无序容器过程中，使得最大复杂因子超过了默认值，则容器会自动增加桶数，并重新进行哈希，以此来减小负载因子的值。需要注意的是，此过程会导致容器迭代器失效，但指向单个键值对的引用或者指针仍然有效。

| 名称              | 说明                                                         | 例子/原型                                                    |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| load_factor()     | 返回容器中当前的负载因子。负载因子，指的是的当前容器中存储键值对的数量（size()）和使用桶数（bucket_count()）的比值，即 load_factor() = size() / bucket_count()。 | float load_factor() const noexcept;                          |
| max_load_factor() | 返回或者设置当前容器的负载因子。                             | get (1)	float max_load_factor() const noexcept;<br/>set (2)	void max_load_factor ( float z ); |
| rehash(n)         | 将当前容器底层使用桶的数量设置为 n。                         | void rehash( size_type n );                                  |
| reserve()         | 将存储桶的数量（也就是 bucket_count() 方法的返回值）设置为至少容纳count个元（不超过最大负载因子）所需的数量，并重新整理容器。 | void reserve ( size_type n );                                |

### 4.2.8 观察者

| 名称            | 说明                                | 例子/原型                                      |
| --------------- | ----------------------------------- | ---------------------------------------------- |
| hash_function() | 返回当前容器使用的哈希函数对象。    | hasher hash_function() const;                  |
| key_eq()        | 返回当前容器使用的key相等函数对象。 | key_equal key_eq() const;                      |
| get_allocator   | 返回用于构造容器的分配器对象。      | allocator_type get_allocator() const noexcept; |

### 4.2.9 非成员函数

| 名称                      | 说明                                                         | 例子/原型                             |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------- |
| operators (unordered_map) | 关系运算符：==、!=                                           |                                       |
| swap (unordered_map)      | 交换容器                                                     | swap(x,y); 等价于 x.swap(y);          |
| begin()                   | 既可以操作容器，也可以操作数组（返回第一个元素指针）。       | begin(myarray);等价于myarray.begin(); |
| end()                     | 既可以操作容器，也可以操作数组（返回后一个元素之后一个位置的指针）。 | end(myarray);等价于myarray.end();     |

## 4.3 自定义无序容器的哈希函数和比较规则

## 4.3.1 自定义哈希函数

简单地理解哈希函数，它可以接收一个元素，并通过内部对该元素做再加工，最终会得出一个整形值并反馈回来。需要注意的是，哈希函数只是一个称谓，其本体并不是普通的函数形式，而是一个函数对象类。因此，如果我们想自定义个哈希函数，就需要自定义一个函数对象类。

```c++
class Person {
public:
    Person(string name, int age) :name(name), age(age) {};
    string getName() const;
    int getAge() const;
private:
    string name;
    int age;
};
string Person::getName() const {
    return this->name;
}
int Person::getAge() const {
    return this->age;
}

class hash_fun {
public:
    int operator()(const Person &A) const {
        return A.getAge();
    }
};

//  容器还无法使用，因为该容器使用的是默认的 std::equal_to<key> 比较规则，但此规则并不适用于该容器。
unordered_set<Person, hash_fun> myset;
```

### 4.3.2 自定义比较规则

std::equal_to\<key>底层实现：

```c++
template<class T>
class equal_to
{
public:   
    bool operator()(const T& _Left, const T& _Right) const{
        return (_Left == _Right);
    }   
};
```

显然，对于我们上面创建的 myset 容器，其内部存储的是 Person 类对象，不支持直接使用 == 运算符做比较。这种情况下，有以下 2 种方式可以解决此问题：

1. 在 Person 类中重载 == 运算符，这会使得 std::equal_to\<key> 比较规则中使用的 == 运算符变得合法，myset 容器就可以继续使用 std::equal_to\<key> 比较规则；
2. 以函数对象类的方式，自定义一个适用于 myset 容器的比较规则。

#### 1) 重载==运算符

```c++
bool operator==(const Person &A, const Person &B) {
    return (A.getAge() == B.getAge());
}

// 最终 myset 容器只会存储 {"zhangsan", 40} 和 {"lisi", 30}
std::unordered_set<Person, hash_fun> myset{ {"zhangsan", 40},{"zhangsan", 40},{"lisi", 40},{"lisi", 30} };
```

#### 2) 以函数对象类的方式自定义比较规则

```c++
class mycmp {
public:
    bool operator()(const Person &A, const Person &B) const {
        return (A.getName() == B.getName()) && (A.getAge() == B.getAge());
    }
};

// 最终该容器内部存有  {"zhangsan", 40}、{"lisi", 40} 和 {"lisi", 30} 这 3 个 Person 对象。
std::unordered_set<Person, hash_fun, mycmp> myset{ {"zhangsan", 40},{"zhangsan", 40},{"lisi", 40},{"lisi", 30} };
```

# 5. STL容器适配器

容器适配器是一个封装了序列容器的类模板，它在一般序列容器的基础上提供了一些不同的功能。之所以称作适配器类，是因为它可以通过适配容器现有的接口来提供不同的功能。

本章将介绍 3 种容器适配器，分别是 stack、queue、priority_queue：

1. stack\<T>：是一个封装了 deque\<T> 容器的适配器类模板，默认实现的是一个后入先出（Last-In-First-Out，LIFO）的压入栈。stack\<T> 模板定义在头文件 stack 中。
2. queue\<T>：是一个封装了 deque\<T> 容器的适配器类模板，默认实现的是一个先入先出（First-In-First-Out，LIFO）的队列。可以为它指定一个符合确定条件的基础容器。queue\<T> 模板定义在头文件 queue 中。
3. priority_queue\<T>：是一个封装了 vector\<T> 容器的适配器类模板，默认实现的是一个会对元素排序，从而保证最大元素总在队列最前面的队列。priority_queue\<T> 模板定义在头文件 queue 中。

| 容器适配器     | 基础容器筛选条件                                             | 默认使用的基础容器 |
| -------------- | ------------------------------------------------------------ | ------------------ |
| stack          | 基础容器需包含以下成员函数：<br/>empty()<br/>size()<br/>back()<br/>push_back()<br/>pop_back()<br/>满足条件的基础容器有 vector、deque、list | deque              |
| queue          | 基础容器需包含以下成员函数：<br/>empty()<br/>size()<br/>front()<br/>back()<br/>push_back()<br/>pop_front()<br/>满足条件的基础容器有 deque、list。 | deque              |
| priority_queue | 基础容器需包含以下成员函数：<br/>empty()<br/>size()<br/>front()<br/>push_back()<br/>pop_back()<br/>满足条件的基础容器有vector、deque。 | vector             |

## 5.1 stack

### 5.1.1 声明初始化

例子：

```c++
#include <stack>
using namespace std;

stack<int> values;
stack<string, list<int>> values; // 底层采用list

list<int> values {1, 2, 3};
stack<int, list<int>> my_stack1(values);
stack<int, list<int>> my_stack(my_stack1);
```

### 5.1.2 成员函数

| 名称      | 说明                                                         | 例子/原型                                                    |
| --------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| empty()   | 当 stack 栈中没有元素时，该成员函数返回 true；反之，返回 false。 | bool empty() const;                                          |
| size()    | 返回 stack 栈中存储元素的个数。                              | size_type size() const;                                      |
| top()     | 返回一个栈顶元素的引用，类型为 T&。如果栈为空，程序会报错。  | reference top();<br/>const_reference top() const;            |
| push(val) | 压入一个元素到栈顶。                                         | void push (const value_type& val);<br/>void push (value_type&& val); |
| emplace() | arg... 可以是一个参数，也可以是多个参数，但它们都只用于构造一个对象，并在栈顶直接生成该对象，作为新的栈顶元素。 | template <class... Args> void emplace (Args&&... args);      |
| pop()     | 弹出栈顶元素。                                               | void pop();                                                  |
| swap()    | 交换容器。                                                   | x.swap(y);                                                   |

### 5.1.3 非成员函数

| 名称                         | 说明                    | 例子/原型                                                    |
| ---------------------------- | ----------------------- | ------------------------------------------------------------ |
| relational operators (stack) | 关系运算法：==、<及其它 | a!=b  等价于 !(a==b)<br/>a>b   等价于   b<a<br/>a<=b 等价于 !(b<a)<br/>a>=b 等价于 !(a<b) |
| swap (stack)                 | 交换容器                | swap(x,y); 等价于 x.swap(y);                                 |

### 5.1.4 非成员类特例化

| 名称                   | 说明 | 例子/原型                                                    |
| ---------------------- | :--- | ------------------------------------------------------------ |
| uses_allocator\<stack> |      | template <class T, class Container, class Alloc><br/>  struct uses_allocator<stack<T,Container>,Alloc><br/>    : uses_allocator<Container,Alloc>::type {} |

```c++
// uses_allocator example
#include <iostream>
#include <memory>
#include <vector>

int main() {
  typedef std::vector<int> Container;
  typedef std::allocator<int> Allocator;

  if (std::uses_allocator<Container,Allocator>::value) {
    Allocator alloc;
    Container foo (5,10,alloc);
    for (auto x:foo) std::cout << x << ' ';
  }
  std::cout << '\n';
  return 0;
} // 10 10 10 10 10
```

## 5.2 queue

### 5.2.1 声明初始化

例子：

```c++
#include <queue>
using namespace std;

queue<int> values;
queue<int, list<int>> values; // 底层采用list

queue<int> values{1,2,3};
queue<int> my_queue(values);
```

### 5.2.2 成员函数

| 名称      | 说明                                                         | 例子/原型                                                    |
| --------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| empty()   | 如果 queue 中没有元素的话，返回 true。                       | bool empty() const;                                          |
| size()    | 返回 queue 中元素的个数。                                    | size_type size() const;                                      |
| front()   | 返回 queue 中第一个元素的引用。如果 queue 是常量，就返回一个常引用；如果 queue 为空，返回值是未定义的。 | reference& front();<br/>const_reference& front() const;      |
| back()    | 返回 queue 中最后一个元素的引用。如果 queue 是常量，就返回一个常引用；如果 queue 为空，返回值是未定义的。 | reference& back();<br/>const_reference& back() const;        |
| push(val) | 在 queue 的尾部添加一个元素。                                | void push (const value_type& val);<br/>void push (value_type&& val); |
| emplace() | 在 queue 的尾部直接添加一个元素。                            | template <class... Args> void emplace (Args&&... args);      |
| pop()     | 删除 queue 中的第一个元素。                                  | void pop();                                                  |
| swap()    | 交换容器。                                                   | x.swap(y);                                                   |

### 5.2.3 非成员函数

| 名称                         | 说明                    | 例子/原型                                                    |
| ---------------------------- | ----------------------- | ------------------------------------------------------------ |
| relational operators (stack) | 关系运算法：==、<及其它 | a!=b  等价于 !(a==b)<br/>a>b   等价于   b<a<br/>a<=b 等价于 !(b<a)<br/>a>=b 等价于 !(a<b) |
| swap (stack)                 | 交换容器                | swap(x,y); 等价于 x.swap(y);                                 |

### 5.2.4 非成员类特例化

| 名称                   | 说明 | 例子/原型                                                    |
| ---------------------- | :--- | ------------------------------------------------------------ |
| uses_allocator\<queue> |      | template <class T, class Container, class Alloc><br/>  struct uses_allocator<queue<T,Container>,Alloc><br/>    : uses_allocator<Container,Alloc>::type {} |

## 5.3 priority_queue

优先级队列，先进队列的元素并不一定先出队列，而是优先级最大的元素最先出队列。

### 5.3.1 声明初始化

```c++
template <typename T,
        typename Container=std::vector<T>,
        typename Compare=std::less<T> >  // 需要比较规则确定优先级
class priority_queue{
    //......
}
```

例子：

```c++
#include <queue>
using namespace std;

priority_queue<int> values;

//使用普通数组
int values[]{4,1,3,2};
priority_queue<int> copy_values(values,values+4);//{4,2,3,1}
// 手动指定底层容器和排序规则
priority_queue<int, deque<int>, greater<int>> copy_values(values, values+4);//{4,2,3,1}
//使用序列式容器
array<int,4> values{ 4,1,3,2};
priority_queue<int> copy_values(values.begin(), values.end());//{4,2,3,1}
```

### 5.3.2 成员函数

和 queue 一样，priority_queue 也没有迭代器，因此访问元素的唯一方式是遍历容器，通过不断移除访问过的元素，去访问下一个元素。

| 名称      | 说明                                                         | 例子/原型                                                    |
| --------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| empty()   | 如果 priority_queue 为空的话，返回 true；反之，返回 false。  | bool empty() const;                                          |
| size()    | 返回 priority_queue 中存储元素的个数。                       | size_type size() const;                                      |
| top()     | 返回 priority_queue 中第一个元素的引用形式                   | reference& top();<br/>const_reference& top() const;          |
| push(val) | 根据既定的排序规则，将元素存储到 priority_queue 中适当的位置。 | void push (const value_type& val);<br/>void push (value_type&& val); |
| emplace() | Args&&... args 表示构造一个存储类型的元素所需要的数据（对于类对象来说，可能需要多个数据构造出一个对象）。此函数的功能是根据既定的排序规则，在容器适配器适当的位置直接生成该新元素。 | template <class... Args> void emplace (Args&&... args);      |
| pop()     | 移除 priority_queue 容器适配器中第一个元素。                 | void pop();                                                  |
| swap()    | 交换容器。                                                   | x.swap(y);                                                   |

### 5.3.3 非成员函数

| 名称         | 说明     | 例子/原型                    |
| ------------ | -------- | ---------------------------- |
| swap (stack) | 交换容器 | swap(x,y); 等价于 x.swap(y); |

### 5.3.4 非成员类特例化

| 名称                            | 说明 | 例子/原型                                                    |
| ------------------------------- | :--- | ------------------------------------------------------------ |
| uses_allocator\<priority_queue> |      | template <class T, class Container, class Compare, class Alloc><br/>  struct uses_allocator<priority_queue<T,Container,Compare>,Alloc><br/>    : uses_allocator<Container,Alloc>::type {} |

### 5.3.5 自定义排序规则

- 自定义函数对象类
- 重载全局 < 或者 > 运算符
- 以友元函数或者成员函数的方式重载 > 或者 < 运算符

# 6. STL迭代器适配器

STL迭代器适配器种类：

| 名称                                                         | 功能                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 反向迭代器（reverse_iterator）                               | 又称“逆向迭代器”，其内部重新定义了递增运算符（++）和递减运算符（--），专门用来实现对容器的逆序遍历。 |
| 安插型迭代器（inserter或者insert_iterator）                  | 通常用于在容器的任何位置添加新的元素，需要注意的是，此类迭代器不能被运用到元素个数固定的容器（比如 array）上。 |
| 流迭代器（istream_iterator / ostream_iterator） 流缓冲区迭代器（istreambuf_iterator / ostreambuf_iterator） | 输入流迭代器用于从文件或者键盘读取数据；相反，输出流迭代器用于将数据输出到文件或者屏幕上。 输入流缓冲区迭代器用于从输入缓冲区中逐个读取数据；输出流缓冲区迭代器用于将数据逐个写入输出流缓冲区。 |
| 移动迭代器（move_iterator）                                  | 此类型迭代器是 C++ 11 标准中新添加的，可以将某个范围的类对象移动到目标范围，而不需要通过拷贝去移动。 |

## 6.1 反向迭代器适配器（reverse_iterator）

值得一提的是，反向迭代器底层可以选用双向迭代器或者随机访问迭代器作为其基础迭代器。不仅如此，通过对 ++（递增）和 --（递减）运算符进行重载，使得：

- 当反向迭代器执行 ++ 运算时，底层的基础迭代器实则在执行 -- 操作，意味着反向迭代器在反向遍历容器；
- 当反向迭代器执行 -- 运算时，底层的基础迭代器实则在执行 ++ 操作，意味着反向迭代器在正向遍历容器。

### 6.1.1 声明初始化

```c++
#include <iterator>
using namespace std;

reverse_iteratorvector<int>::iterator> my_reiter; // 不指向任何对象的反向迭代器

vector<int> myvector{1,2,3,4,5};
// 创建并初始化，end()向左移一位，my_reiter1指向最后一个元素
reverse_iterator<vector<int>::iterator> my_reiter1(myvector.end());
// 创建并初始化，begin()向左移一位，my_reiter2指向第一个元素前一个元素
reverse_iterator<vector<int>::iterator> my_reiter2(myvector.begin());

// 复制或拷贝
reverse_iterator<vector<int>::iterator> my_reiter(myvector.rbegin());
auto my_reiter = myvector.rbegin();
```

### 6.1.2 成员函数

| 重载运算符/函数 | 功能                                                         |
| --------------- | ------------------------------------------------------------ |
| operator*       | 以引用的形式返回当前迭代器指向的元素。                       |
| operator+       | 返回一个反向迭代器，其指向距离当前指向的元素之后 n 个位置的元素。此操作要求基础迭代器为随机访问迭代器。 |
| operator++      | 重载前置 ++ 和后置 ++ 运算符。                               |
| operator+=      | 当前反向迭代器前进 n 个位置，此操作要求基础迭代器为随机访问迭代器。 |
| operator-       | 返回一个反向迭代器，其指向距离当前指向的元素之前 n 个位置的元素。此操作要求基础迭代器为随机访问迭代器。 |
| operator--      | 重载前置 -- 和后置 -- 运算符。                               |
| operator-=      | 当前反向迭代器后退 n 个位置，此操作要求基础迭代器为随机访问迭代器。 |
| operator->      | 返回一个指针，其指向当前迭代器指向的元素。                   |
| operator[n]     | 访问和当前反向迭代器相距 n 个位置处的元素。                  |
| base()          | 返回当前反向迭代器底层所使用的基础迭代器                     |

### 6.1.3 非成员函数重载

| 名称                 | 说明       | 例子/原型                                       |
| -------------------- | ---------- | ----------------------------------------------- |
| relational operators | 关系运算符 | ==、!=、<、<=、>、>=                            |
| operator+            | 加法运算符 | auto rev_it = 3 + myvector.rbegin();            |
| operator-            | 减法运算符 | auto ret = myvector.rend() - myvector.rbegin(); |

## 6.2 插入迭代器适配器（insert_iterator）

插入迭代器适配器种类：

| 迭代器适配器          | 功能                                                         |
| --------------------- | ------------------------------------------------------------ |
| back_insert_iterator  | 在指定容器的尾部插入新元素，但前提必须是提供有 push_back() 成员方法的容器（包括 vector、deque 和 list）。 |
| front_insert_iterator | 在指定容器的头部插入新元素，但前提必须是提供有 push_front() 成员方法的容器（包括 list、deque 和 forward_list）。 |
| insert_iterator       | 在容器的指定位置之前插入新元素，前提是该容器必须提供有 insert() 成员方法。**因此 insert_iterator 是唯一可用于关联式容器的插入迭代器。** |

### 6.2.1 声明初始化

```c++
#include <iterator>
using namespace std;

vector<int> foo;
// 创建一个可向 foo 容器尾部添加新元素的迭代器
back_insert_iterator<vector<int>> back_it(foo);

forward_list<int> foo;
// 创建一个可向 foo 容器头部添加新元素的迭代器
front_insert_iterator<forward_list<int>> front_it(foo);

// 创建insert_iterator，第二个参数是一个迭代器指向插入位置
insert_iterator<vector<int>> insert_it(foo, foo.begin() + 1);

// 使用迭代器生成器
auto back_it = back_inserter(foo);
auto front_it = front_inserter(foo);
auto insert_it = inserter(foo, foo.begin() + 1);
```

### 6.2.2 成员函数

| 重载运算符 | 功能                                                         |
| ---------- | ------------------------------------------------------------ |
| operator=  | 在容器末尾插入一个新元素，并使用实参初始化它。<br/>在容器的开头插入一个新元素，用实参初始化它。<br/>将新元素插入容器中构造时传递的迭代器所指向的位置，用实参初始化新元素。 |
| operator*  | 什么也不做。返回对象的引用。                                 |
| operator++ | 什么也不做。返回对象的引用。                                 |

### 6.2.3 例子

back_insert_iterator：

```c++
#include <iostream>
#include <iterator>
#include <vector>
using namespace std;
int main() {
    //创建一个 vector 容器
    std::vector<int> foo;
    //创建一个可向 foo 容器尾部添加新元素的迭代器
    std::back_insert_iterator< std::vector<int> > back_it(foo);
    //将 5 插入到 foo 的末尾
    back_it = 5;
    //将 4 插入到当前 foo 的末尾
    back_it = 4;
    //将 3 插入到当前 foo 的末尾
    back_it = 3;
    //将 6 插入到当前 foo 的末尾
    back_it = 6;
    //输出 foo 容器中的元素：5 4 3 6
    for (std::vector<int>::iterator it = foo.begin(); it != foo.end(); ++it)
        std::cout << *it << ' ';
    return 0;
}
```

front_insert_iterator：

```c++
#include <iostream>
#include <iterator>
#include <forward_list>
using namespace std;
int main() {
    //创建一个 forward_list 容器
    std::forward_list<int> foo;
    //创建一个前插入迭代器
    //std::front_insert_iterator< std::forward_list<int> > front_it(foo);
    std::front_insert_iterator< std::forward_list<int> > front_it = front_inserter(foo);
    //向 foo 容器的头部插入元素
    front_it = 5;
    front_it = 4;
    front_it = 3;
    front_it = 6; // 6 3 4 5
    for (std::forward_list<int>::iterator it = foo.begin(); it != foo.end(); ++it)
        std::cout << *it << ' ';
    return 0;
}
```

insert_iterator：

```c++
#include <iostream>
#include <iterator>
#include <list>
using namespace std;
int main() {
    //初始化为 {5,5}
    std::list<int> foo(2,5);
    //定义一个基础迭代器，用于指定要插入新元素的位置
    std::list<int>::iterator it = ++foo.begin();
    //创建一个 insert_iterator 迭代器
    //std::insert_iterator< std::list<int> > insert_it(foo, it);
    std::insert_iterator< std::list<int> > insert_it = inserter(foo, it);
    //向 foo 容器中插入元素
    insert_it = 1;
    insert_it = 2;
    insert_it = 3;
    insert_it = 4;
    //输出 foo 容器存储的元素 5 1 2 3 4 5
    for (std::list<int>::iterator it = foo.begin(); it != foo.end(); ++it)
        std::cout << *it << ' ';
    return 0;
}
```

## 6.3 输入/输出流迭代器

流缓冲区迭代器会将元素以字符的形式（包括 char、wchar_t、char16_t 及 char32_t 等）读或者写到流缓冲区中，由于不会涉及数据类型的转换，读写数据的速度比流迭代器要快。

| 迭代器              | 成员运算符        | 非成员运算符 | 说明                                          |
| ------------------- | ----------------- | ------------ | --------------------------------------------- |
| istream_iterator    | ++p、p++、*p、p-> | ==、!=       | 重载 ++ 运算符内部会调用`operator >>`读取数据 |
| ostream_iterator    | ++p、p++、*p、p=  |              | 重载赋值（=）运算符实现                       |
| istreambuf_iterator | ++p、p++、*p、p-> | ==、!=       | 重载 ++ 运算符内部会调用`operator >>`读取数据 |
| ostreambuf_iterator | ++p、p++、*p、p=  |              | 重载赋值（=）运算符实现                       |

### 6.3.1 声明初始化

```c++
#include <iterator>
using namespace std;

// 创建了一个可读取 double 类型元素，并代表结束标志的输入流迭代器
istream_iterator<double> eos;
// 创建了一个可读取 char 字符，并代表结束标志的输入流缓冲区迭代器
istreambuf_iterator<char> end_in;

istream_iterator<double> in_it(cin);
istream_iterator<double> in_it2(in_it);

istreambuf_iterator<char> inbuf_it(cin);
// rdbuf() 获取指定流缓冲区的地址，生成的迭代器相同
istreambuf_iterator<char> inbuf_it(cin.rdbuf());

//--------------------------------------

// 用cout初始化并设置分隔符
ostream_iterator<int> out_it(cout，",");
// 拷贝
ostream_iterator<int> out_it2(out_it);

ostreambuf_iterator<char> outbuf_it(cout);
// rdbuf() 获取指定流缓冲区的地址，生成的迭代器相同
ostreambuf_iterator<char> outbuf_it(cout.rdbuf());
```

### 6.3.2 例子

istream_iterator：

```c++
#include <iostream>
#include <iterator>
using namespace std;
int main() {
    //用于接收输入流中的数据
    double value1, value2;
    cout << "请输入 2 个小数: ";
    //创建表示结束的输入流迭代器
    istream_iterator<double> eos;
    //创建一个可逐个读取输入流中数据的迭代器，同时这里会让用户输入数据
    istream_iterator<double> iit(cin);
    //判断输入流中是否有数据
    if (iit != eos) {
        //读取一个元素，并赋值给 value1
        value1 = *iit;
    }
    //如果输入流中此时没有数据，则用户要输入一个；反之，如果流中有数据，iit 迭代器后移一位，做读取下一个元素做准备
    iit++;
    if (iit != eos) {
        //读取第二个元素，赋值给 value2
        value2 = *iit;
    }
    //输出读取到的 2 个元素
    cout << "value1 = " << value1 << endl;
    cout << "value2 = " << value2 << endl;
    return 0;
}
```

```powershell
请输入 2 个小数: 1.2 2.3
value1 = 1.2
value2 = 2.3
```

注意，只有读取到 EOF 流结束符时，程序中的 iit 才会和 eos 相等。另外，Windows 平台上使用 Ctrl+Z 组合键输入 ^Z 表示 EOF 流结束符，此结束符需要单独输入，或者输入换行符之后再输入才有效。

ostream_iterator：

```c++
#include <iostream>
#include <iterator>
#include <vector>
#include <algorithm>    // std::copy
using namespace std;
int main() {
    //创建一个 vector 容器
    vector<int> myvector;
    //初始化 myvector 容器
    for (int i = 1; i < 10; ++i) {
        myvector.push_back(i);
    }
    //创建输出流迭代器
    std::ostream_iterator<int> out_it(std::cout, ", ");
    //将 myvector 容器中存储的元素写入到 cout 输出流中
    std::copy(myvector.begin(), myvector.end(), out_it);
    return 0;
} //1, 2, 3, 4, 5, 6, 7, 8, 9, 
```

istreambuf_iterator：

```c++
#include <iostream>     // std::cin, std::cout
#include <iterator>     // std::istreambuf_iterator
#include <string>       // std::string
using namespace std;
int main() {
    //创建结束流缓冲区迭代器
    istreambuf_iterator<char> eos;
    //创建一个从输入缓冲区读取字符元素的迭代器
    istreambuf_iterator<char> iit(cin);
    string mystring;
    cout << "向缓冲区输入元素：\n";
    //不断从缓冲区读取数据，直到读取到 EOF 流结束符
    while (iit != eos) {
        mystring += *iit++;
    }
    cout << "string：" << mystring;
    return 0;
}
```

```powershell
向缓冲区输入元素：
abc ↙
^Z ↙
string：abc
```

ostreambuf_iterator：

```c++
#include <iostream>     // std::cin, std::cout
#include <iterator>     // std::ostreambuf_iterator
#include <string>       // std::string
#include <algorithm>    // std::copy
int main() {
    //创建一个和输出流缓冲区相关联的迭代器
    std::ostreambuf_iterator<char> out_it(std::cout); // stdout iterator
    //向输出流缓冲区中写入字符元素
    *out_it = 'S';
    *out_it = 'T';
    *out_it = 'L';
    //和 copy() 函数连用
    std::string mystring("\nhttp://c.biancheng.net/stl/");
    //将 mystring 中的字符串全部写入到输出流缓冲区中
    std::copy(mystring.begin(), mystring.end(), out_it);
    return 0;
}
```

```powershell
STL
http://c.biancheng.net/stl/
```

## 6.4 移动迭代器(move_iterator)

### 6.4.1 声明初始化

注意，基础迭代器的类型虽然没有明确要求，但该模板类中某些成员方法的底层实现，需要此基础迭代器为双向迭代器或者随机访问迭代器。也就是说，如果指定的 Iterator 类型仅仅是输入迭代器，则某些成员方法将无法使用。

```c++
#include <iterator>
using namespace std;

//将 vector 容器的随机访问迭代器作为新建移动迭代器底层使用的基础迭代器
typedef vector<string>::iterator Iter;
//调用默认构造函数，创建一个不指向任何对象的移动迭代器
move_iterator<Iter> mIter;

vector<string> myvec{ "one","two","three" };
//将 vector 容器的随机访问迭代器作为新建移动迭代器底层使用的基础迭代器
typedef vector<string>::iterator Iter;
//创建并初始化移动迭代器
move_iterator<Iter> mIter(myvec.begin());
move_iterator<Iter> mIter2(mIter);

//将 make_move_iterator() 的返回值赋值给同类型的 mIter 迭代器
auto mIter = make_move_iterator(myvec.begin());
```

### 6.4.2 成员函数

| 重载运算符/函数 | 功能                                                         |
| --------------- | ------------------------------------------------------------ |
| operator*       | 以引用的形式返回当前迭代器指向的元素。                       |
| operator+       | 返回一个移动迭代器，其指向距离当前指向的元素之后 n 个位置的元素。此操作要求基础迭代器为随机访问迭代器。 |
| operator++      | 重载前置 ++ 和后置 ++ 运算符。                               |
| operator+=      | 当前移动迭代器前进 n 个位置，此操作要求基础迭代器为随机访问迭代器。 |
| operator-       | 返回一个移动迭代器，其指向距离当前指向的元素之前 n 个位置的元素。此操作要求基础迭代器为随机访问迭代器。 |
| operator--      | 重载前置 -- 和后置 -- 运算符。                               |
| operator-=      | 当前移动迭代器后退 n 个位置，此操作要求基础迭代器为随机访问迭代器。 |
| operator->      | 返回一个指针，其指向当前迭代器指向的元素。                   |
| operator[n]     | 访问和当前移动迭代器相距 n 个位置处的元素。                  |
| base()          | 返回当前移动迭代器底层所使用的基础迭代器                     |

### 6.4.3 非成员函数重载

| 名称                 | 说明       | 例子/原型                |
| -------------------- | ---------- | ------------------------ |
| relational operators | 关系运算符 | ==、!=、<、<=、>、>=     |
| operator+            | 加法运算符 | auto rev_it = 3 + mIter; |
| operator-            | 减法运算符 | auto ret = lhs - rhs;    |

### 6.4.4 例子

```c++
#include <iostream>
#include <vector>
#include <list>
#include <string>
using namespace std;
int main()
{
    //创建并初始化一个 vector 容器
    vector<string> myvec{ "STL","Python","Java" };
    //再次创建一个 vector 容器，利用 myvec 为其初始化
    vector<string> othvec(make_move_iterator(myvec.begin()), make_move_iterator(myvec.end()));
   
    cout << "myvec:" << endl;
    //输出 myvec 容器中的元素
    for (auto ch : myvec) {
        cout << ch << " ";
    }
    cout << endl << "othvec:" << endl;
    //输出 othvec 容器中的元素
    for (auto ch : othvec) {
        cout << ch << " ";
    }
    return 0;
}
```

```powershell
myvec:

othvec:
STL Python Java
```

## 6.5 迭代器辅助函数

| 迭代器辅助函数        | 功能                                                         | 说明                                                         |
| --------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| advance(it, n)        | it 表示某个迭代器，n 为整数。该函数的功能是将 it 迭代器前进或后退 n 个位置。不会检测 it 迭代器移动 n 个位置的可行性 | it 为随机访问迭代器时，底层采用 it+n 操作实现；it 为其他类型迭代器时，通过重复执行 n 个 ++ 或者 -- 操作实现。 |
| distance(first, last) | 该函数的功能是计算 first 和 last 之间的距离，first 和 last 都是迭代器，可以是输入迭代器、前向迭代器、双向迭代器以及随机访问迭代器。 | 同上，根据迭代器类型实现机制不同。                           |
| begin(cont)           | cont 表示某个容器，该函数可以返回一个指向 cont 容器中第一个元素的迭代器。 | 可以处理容器和数组(此时返回指针)                             |
| end(cont)             | cont 表示某个容器，该函数可以返回一个指向 cont 容器中最后一个元素之后位置的迭代器。 | 可以处理容器和数组(此时返回指针)                             |
| prev(it, n=1)         | it 为指定的迭代器，该函数默认可以返回一个指向上一个位置处的迭代器。注意，it 至少为双向迭代器。不会检测 it 迭代器移动 n 个位置的可行性。 | advance修改it，此函数不会，返回移动后的迭代器。              |
| next(it, n=1)         | it 为指定的迭代器，该函数默认可以返回一个指向下一个位置处的迭代器。注意，it 最少为前向迭代器。不会检测 it 迭代器移动 n 个位置的可行性。 | advance修改it，此函数不会，返回移动后的迭代器。              |

## 6.6 const_iterator转换为iterator

const_iterator转iterator不能使用const_cast，因为它们是不同的类。

```c++
// cont：某个容器
// iter：初始化为指向容器第一个元素的iterator/reverse_iterator迭代器
// citer：要转换的常量迭代器

//将 const_iterator 转换为 iterator
advance(iter, distance<cont<T>::const_iterator>(iter,citer));
//将 const_reverse_iterator 转换为 reverse_iterator
advance(iter, distance<cont<T>::const_reverse_iterator>(iter,citer));
```

该实现方式的本质是，先创建一个迭代器 iter，并将其初始化为指向容器中第一个元素的位置。在此基础上，通过计算和目标迭代器 citer 的距离（调用 distance()），将其移动至和 citer同一个位置（调用 advance()），由此就可以间接得到一个指向同一位置的 iter 迭代器。

注意，在使用 distance() 函数时，必须额外指明 2 个参数为 const 迭代器类型，否则会因为传入的 iter 和 citer 类型不一致导致 distance() 函数编译出错。

# 7. algorithm

当某个 STL 容器提供有和算法同名的成员方法时，应该使用哪一个呢？大多数情况下，我们应该使用 STL 容器提供的成员方法，而不是同名的 STL 算法，原因包括：

1. 虽然同名，但它们的底层实现并不完全相同。相比同名的算法，容器的成员方法和自身结合地更加紧密。
2. 相比同名的算法，STL 容器提供的成员方法往往执行效率更高；

自定义STL算法规则，应优先使用函数对象，其效率更高：

```c++
#include <iostream>     // std::cout
#include <algorithm>    // std::sort
#include <vector>       // std::vector
//以普通函数的方式实现自定义排序规则
inline bool mycomp(int i, int j) {
    return (i < j);
}
//以函数对象的方式实现自定义排序规则
class mycomp2 {
public:
    bool operator() (int i, int j) {
        return (i < j);
    }
};
int main() {
    std::vector<int> myvector{ 32, 71, 12, 45, 26, 80, 53, 33 };
    //调用普通函数定义的排序规则
    std::sort(myvector.begin(), myvector.end(), mycomp);
    //调用函数对象定义的排序规则
    //std::sort(myvector.begin(), myvector.end(), mycomp2());
   
    for (std::vector<int>::iterator it = myvector.begin(); it != myvector.end(); ++it) {
        std::cout << *it << ' ';
    }
    return 0;
}
```

## 7.1 不修改序列的操作

| 名称                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| bool **all_of**(first, last, pred);                          | 返回 true，前提是序列[first, last)中的元素都可以使谓词pred返回 true，范围为空返回 true。 |
| bool **any_of**(first, last, pred);                          | 返回 true，前提是序列[first, last)中的任意一个元素可以使谓词pred返回 true，范围为空返回false。 |
| bool **none_of**(first, last, pred);                         | 返回 true，前提是序列[first, last)中没有元素可以使谓词pred返回 true，范围为空返回 true。 |
| Function **for_each**(first, last, fn);                      | 返回 fn，对范围[first,last)中的每个元素应用函数fn。          |
| InputIterator **find**(first, last, val);                    | 返回范围[first,last)中值等于val的第一个元素的迭代器。如果没有找到这样的元素，函数返回last。 |
| InputIterator **find_if**(first, last, pred);                | 返回范围[first,last)中满足pred的第一个元素的迭代器。如果没有找到这样的元素，函数返回last。 |
| InputIterator **find_if_not**(first, last, pred);            | 返回范围[first,last)中不满足pred的第一个元素的迭代器。如果没有找到这样的元素，函数返回last。 |
| ForwardIterator **find_end**(first1, last1, first2, last2[, pred]); | 在[first1,last1)中查找[first2,last2)最后一次出现的位置。如果没有找到这样的元素，函数返回last1。pred谓词比对两个序列中的元素是否相等。 |
| ForwardIterator **find_first_of**(first1, last1, first2, last2[, pred]); | 在[first1,last1)中查找[first2,last2)任一元素第一次出现的位置。如果没有找到这样的元素，函数返回last1。pred谓词比对两个序列中的元素是否相等。 |
| ForwardIterator **adjacent_find**(first, last[, pred]);      | 在[first,last)中查找两个连续元素的第一次出现，并返回第一个的迭代器。如果没有找到这样的对，则返回last。 |
| **count**(first, last, val);                                 | 返回范围[first,last)中等于val的元素个数。                    |
| **count_if**(first, last, pred);                             | 返回范围[first,last)中满足pred的元素个数。                   |
| pair<iter1, iter2> **mismatch**(first1, last1, first2[, pred]);<br/>pair<iter1, iter2> **mismatch**(first1, last1, first2, last2[, pred]); | 序列不相等的位置<br/>(1)比较范围[first1,last1)中的元素与从first2开始的范围中的元素，并返回两个序列中第一个不匹配的元素。如果第二个序列比第一个序列短，结果是未定义的。<br/>(2)比较范围[first1,last1)与范围[first2,last2)，返回两个序列中第一个不匹配的元素。<br/>(3)pred谓词可选，用于比对两个序列中的元素是否相等。 |
| bool **equal**(first1, last1, first2[, pred]);<br/>bool **equal**(first1, last1, first2, last2[, pred]); | 比较序列是否相等<br>(1)比较范围[first1,last1)与从first2开始的范围。如果第二个序列比第一个序列短，结果是未定义的。<br/>(2)比较范围[first1,last1)与范围[first2,last2)，长度不相等必定返回false。<br/>(3)pred谓词可选，用于比对两个序列中的元素是否相等。 |
| bool **is_permutation**(first1, last1, first2[, pred]);<br/>bool **is_permutation**(first1, last1, first2, last2[, pred]); | 判断一个序列是否为另一个序列的排列<br/>(1)比较范围[first1,last1)与从first2开始的范围。如果第二个序列比第一个序列短，结果是未定义的。<br/>(2)比较范围[first1,last1)与范围[first2,last2)，长度不相等必定返回false。<br/>(3)pred谓词可选，用于比对两个序列中的元素是否相等。 |
| iter1 **search**(first1, last1, first2, last2[, pred]);      | 在[first1,last1)中查找[first2,last2)第一次出现的位置。如果没有找到这样的元素，函数返回last1。pred谓词比对两个序列中的元素是否相等。区别于**find_end**。 |
| iter **search_n**(first, last, count, val[, pred]);          | 在[first,last)中查找count个val出现的位置。如果没有找到，函数返回last。pred谓词比对两个序列中的元素是否相等。 |

## 7.2 修改序列操作

| 名称                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| OutputIterator **copy**(first, last, result);                | 将范围[first,last)中的元素复制到从result开始的范围中。返回：指向已复制元素的目标范围末尾的迭代器。 |
| OutputIterator **copy_n**(first, last, result);              | 将first开始的范围中的前n个元素复制到result开始的范围中。返回：指向已复制元素的目标范围末尾的迭代器。 |
| OutputIterator **copy_if**(first, last, result, pred);       | 将范围[first,last)中满足pred的元素复制到从result开始的范围中。返回：指向已复制元素的目标范围末尾的迭代器。 |
| BidirectionalIterator **copy_backward**(first, last, result); | 将范围[first,last)中的元素复制到以result结尾(不包含result)的范围中，元素顺序不变。返回：返回指向目标范围内第一个元素的迭代器。 |
| OutputIterator **move**(first, last, result);                | 将范围[first,last)中的元素移动到从result开始的范围中。调用之后，范围[first,last)中的元素保持未指定但有效的状态。范围不能重叠，result不能指向[first,last)范围内的元素。 |
| BidirectionalIterator **move_backward**(first, last, result); | 移动而非拷贝，参考copy_backward                              |
| void **swap**(a, b);                                         | 交换序列或数组，从\<algorithm>迁移到\<utility>               |
| ForwardIterator2 **swap_ranges**(first1, last1, first2);     | 按范围交换容器，返回序列2最后一个被交换元素的下一个位置。    |
| void **iter_swap**(a, b);                                    | 交换a和b迭代器指向的值。                                     |
| OutputIterator **transform**(first1, last1, result, op);<br/>OutputIterator **transform**(first1, last1, first2, result, binary_op); | (1)将[first,last)中的元素执行op(a)后，存入result开始的序列，返回的迭代器指向输出序列所保存的最后一个元素的下一个位置。<br/>(2)将[first,last)中的元素与first2开始的元素执行op(a, b)后，存入result开始的序列，返回的迭代器指向输出序列所保存的最后一个元素的下一个位置。 |
| void **replace**(first, last, old_value, new_value);         | 将[first,last)范围内的所有old_value替换为new_value。         |
| void **replace_if**(first, last, pred, new_value);           | 将[first,last)范围内的所有满足pred的值替换为new_value。      |
| OutputIterator **replace_copy**(first, last, result, old_value, new_value); | 将[first,last)范围内的所有old_value替换为new_value，并拷贝到result开始的序列。 |
| OutputIterator **replace_copy_if**(first, last, result, pred, new_value); | 将[first,last)范围内的所有满足pred的值替换为new_value，并拷贝到result开始的序列。 |
| void **fill**(first, last, val);                             | 将范围[first,last)填充val。                                  |
| OutputIterator **fill_n**(first, n, val);                    | 从first填充n个val，返回：指向最后一个填充元素之后的元素的迭代器。 |
| void **generate**(first, last, gen);                         | 用gen生成值并填充到[first,last)中。                          |
| void **generate_n**(first, n, gen);                          | 用gen生成n个值并填充到first开始的序列中。                    |
| ForwardIterator **remove**(first, last, val);                | 移除[first,last)中所有值等于val的元素，返回新序列最后一个元素之后位置的迭代器。基本上每个元素都是通过用它后面的元素覆盖它来实现移除的。 |
| ForwardIterator **remove_if**(first, last, pred);            | 移除[first,last)中所有满足pred的元素，返回新序列最后一个元素之后位置的迭代器。 |
| OutputIterator **remove_copy**(first, last, result, val);    | 移除[first,last)中所有值等于val的元素并存入result开始的序列，返回一个指向最后一个被复制到目的序列的元素的后一个位置的迭代器。 |
| OutputIterator **remove_copy_if**(first, last, result, pred); | 移除[first,last)中所有满足pred的元素并存入result开始的序列，返回一个指向最后一个被复制到目的序列的元素的后一个位置的迭代器。 |
| ForwardIterator **unique**(first, last[, pred]);             | 删除[first,last)中的连续重复元素只保留一个，返回最后一个未移除元素后面的元素的迭代器。谓词pred可选。 |
| OutputIterator **unique_copy**(first, last, result[, pred]); | 删除[first,last)中的连续重复元素只保留一个，并存入result开始的序列，返回result中最后一个未移除元素后面的元素的迭代器。谓词pred可选。 |
| void **reverse**(first, last);                               | 反转[first,last)范围内元素的顺序。                           |
| OutputIterator **reverse_copy**(first, last, result);        | 反转后存入result开始的序列，指向复制范围末端的输出迭代器。   |
| ForwardIterator **rotate**(first, middle, last);             | 旋转[first,last)范围内的元素，使middle成为新的首元素，返回旧的首元素的迭代器。 |
| OutputIterator **rotate_copy**(first, middle, last, result); | 旋转[first,last)范围内的元素，使middle成为新的首元素，并存入result开始的序列，返回最后一个拷贝元素后一个元素的迭代器。 |
| void **random_shuffle**(first, last[, gen]);                 | 将[first,last)范围内的元素随机打乱，提供gen时按照gen的规则打乱。 |
| void **shuffle**(first, last, g);                            | 随机地重新排列范围[first,last]中的元素，使用g作为均匀随机数生成器。 |

## 7.3 分区

| 名称                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| bool **is_partitioned**(first, last, pred);                  | **测试范围是否已分区**<br/>如果范围[first,last)中所有pred返回true的元素都在pred返回false的元素之前，则返回true。如果范围为空，函数返回true。 |
| ForwardIterator **partition**(first, last, pred);            | **将范围分成两个区间**<br/>重新排列范围[first,last)中的元素，使pred返回true的所有元素都在它返回false的所有元素之前。迭代器返回指向第二组第一个元素的点。每个组内的相对顺序不一定与呼叫前相同(不稳定)。 |
| BidirectionalIterator **stable_partition**(first, last, pred); | **稳定的将范围分成两个区间**<br/>重新排列范围[first,last)中的元素，使pred返回true的所有元素都在它返回false的所有元素之前。迭代器返回指向第二组第一个元素的点。并且与partition函数不同的是，保留了每个组中元素的相对顺序。 |
| pair<OutIter1,OutIter2> **partition_copy**(first, last, result_true, result_false, pred); | **分组拷贝**<br/>范围[first,last)中的元素，使pred返回true则复制到result_true所指向的范围内，OutIter1指定复制的末尾；将不返回true的元素复制到result_false所指向的范围内，OutIter2指定复制的末尾。 |
| ForwardIterator **partition_point**(first, last, pred);      | **获取分区点**<br/>返回一个指向分区范围[first,last)中pred不为true的第一个元素的迭代器，指示其分区点。范围内的元素应该已经被分区了，就好像分区是用相同的参数调用的一样。 |

## 7.4 排序

| 名称                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| void **sort**(first, last[, comp]);                          | **不稳定排序**<br/>将范围[first,last)中的元素按升序排序，comp指定其它排序规则。 |
| void **stable_sort**(first, last[, comp]);                   | **稳定排序**<br/>将范围[first,last)中的元素按升序排序，comp指定其它排序规则。 |
| void **partial_sort**(first, middle, last[, comp]);          | **部分排序**<br/>在范围[first,last)中选最小元素有序放入[first,middle)，其余元素没有特定的顺序，comp指定其它排序规则。 |
| RandomAccessIterator **partial_sort_copy**(first, last, result_first, result_last[, comp]); | **部分排序拷贝**<br/>在范围[first,last)中选最小元素有序放入[result_first,result_last)，原序列不变，comp指定其它排序规则。返回一个迭代器，指向在结果序列中写入的最后一个元素后面的元素，当序列2比序列1大时有用。 |
| bool **is_sorted**(first, last[, comp]);                     | **是否排序**                                                 |
| ForwardIterator **is_sorted_until**(first, last[, comp]);    | **查找范围内第一个未排序的元素**<br/>返回指向范围[first,last)中第一个不按升序排列的元素的迭代器。 |
| void **nth_element**(first, nth, last[, comp]);              | **第n小元素**<br/>在范围[first,last)中找到第n小元素，放到nth所在位置，并且nth左边的元素比其小，右边元素比其大。comp指定其它排序规则。 |

## 7.5 二分查找

| 名称                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ForwardIterator **lower_bound**(first, last, val[, comp]);   | **返回迭代器的左边界**<br/>返回范围[first,last)中第一个大于等于val的元素的迭代器，没有找到返回last。 |
| ForwardIterator **upper_bound**(first, last, val[, comp]);   | **返回迭代器的右边界**<br/>返回范围[first,last)中第一个大于val的元素的迭代器，没有找到返回last。 |
| pair<Iter1,Iter2> **equal_range**(first, last, val[, comp]); | **得到相等元素的子范围**<br/>[Iter1,Iter2)中的元素等于val，这些值与函数lower_bound和upper_bound分别返回的值相同。 |
| bool binary_search(first, last, val[, comp]);                | **二分查找**<br/>[first,last)必须有序，二分查找确定是否包涵val。 |

## 7.6 合并

| 名称                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| OutputIterator **merge**(first1, last1, first2, last2, result[, comp]); | **合并两个排序**<br/>[first1,last1)和[first2,last2)分别有序，合并到result中。归并排序的合并过程。 |
| void **inplace_merge**(first, middle, last[, comp]);         | **合并两个区间**<br/>[first,middle)和[middle,last)分别有序，原地合并。 |
| bool **includes**(first1, last1, first2, last2[, comp]);     | **测试排序范围是否包含另一个排序范围**<br/>如果排序范围[first1,last1)包含排序范围[first2,last2)中的所有元素，则返回true。 |
| OutputIterator **set_union**(first1, last1, first2, last2, result[, comp]); | **两个排序范围的并集**<br/>用两个排序范围[first1,last1)和[first2,last2)的集合并集构造一个由result指向的位置开始的排序范围。两个集合的并集是由存在于其中一个集合或两个集合中的元素构成的。在第一个范围中具有相等元素的第二个范围中的元素不会复制到结果范围。 |
| OutputIterator **set_intersection**(first1, last1, first2, last2, result[, comp]); | **两个排序范围的交集**<br/>构造一个从result所指向的位置开始的排序范围，并使用两个排序范围[first1,last1)和[first2,last2)的集合交集。两个集合的交集只由两个集合中都存在的元素构成。函数复制的元素总是来自第一个范围，顺序相同。 |
| OutputIterator **set_difference**(first1, last1, first2, last2, result[, comp]); | **两个排序范围的差集**<br/>两个集合的差集是由第一个集合中存在的元素构成的，而第二个集合中没有。函数复制的元素总是来自第一个范围，顺序相同。对于支持一个值多次出现的容器，其差值包括与第一个范围中给定值相同的出现次数，减去第二个范围中匹配元素的数量，保持顺序。 |
| OutputIterator **set_symmetric_difference**(first1, last1, first2, last2, result[, comp]); | **两个排序范围的对称差**<br/>两个集合的对称差是由在其中一个集合中存在而在另一个集合中不存在的元素构成的。在每个范围中的等价元素中，被丢弃的是那些在调用之前以存在顺序出现的元素。复制的元素也保留现有的顺序。 |

## 7.7 堆

标准容器适配器priority_queue自动调用make_heap、push_heap和pop_heap来维护容器的堆属性。

| 名称                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| void **push_heap**(first, last[, comp]);                     | **将元素推入堆范围**<br/>给定范围为[first,last-1)的堆，此函数通过将(last-1)中的值放到其中的相应位置，将认为是堆的范围扩展到[first,last)。<br/>可以通过调用make_heap将范围组织成堆。在此之后，如果分别使用push_heap和pop_heap向其添加和删除元素，则会保留其堆属性。 |
| void **pop_heap**(first, last[, comp]);                      | **从堆范围弹出元素**<br/>重新排列堆范围[first,last)中的元素，使堆的长度减1：具有最高值的元素移动到(last-1)。虽然具有最高值的元素从first移动到(last-1)(现在已经不在堆中)，但其他元素将按照以下方式重新组织:范围[first,last-1)保留堆的属性。 |
| void **make_heap**(first, last[, comp]);                     | **从范围内生成堆**<br/>以形成堆的方式重新排列范围[first,last)中的元素。<br/>堆是一种组织范围内元素的方法，它允许在任何时刻快速检索值最高的元素(使用pop_heap)，甚至是重复检索，同时允许快速插入新元素(使用push_heap)。<br/>具有最高值的元素总是被first指向。其他元素的顺序取决于特定的实现，但在此头文件的所有堆相关函数中是一致的。 |
| void **sort_heap**(first, last[, comp]);                     | **对堆的元素排序**<br/>将堆范围[first,last)中的元素按升序排序。范围失去了作为堆的属性。 |
| void **is_heap**(first, last[, comp]);                       | **测试是否为堆**<br/>如果范围[first,last)形成一个堆，返回true，就像用make_heap构造一样。 |
| RandomAccessIterator **is_heap_until**(first, last[, comp]); | **找到第一个不是堆顺序的元素**<br/>从first到返回的迭代器之间的范围是堆。 |

## 7.8 最大/最小

| 名称                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| **min**(a, b[, comp]);<br/>**min**(il[, comp]);              | **返回最小值**<br/>返回a和b中最小的一个。如果两者相等，则返回a。<br/>il为初始化列表，返回列表中所有元素中最小的一个。如果它们多于一个，返回第一个。 |
| **max**(a, b[, comp]);<br/>**max**(il[, comp]);              | **返回最大值**<br/>返回a和b中最大的一个。如果两者相等，则返回a。<br/>il为初始化列表，返回列表中所有元素中最大的一个。如果它们多于一个，返回第一个。 |
| pair<T,T> **minmax**(a, b[, comp]);<br/>pair<T,T> **minmax**(il[, comp]); | **返回最小和最大值**<br/>pair中第一个元素是最小值，第二个元素是最大值。 |
| ForwardIterator **min_element**(first, last[, comp]);        | **返回返回中最小元素的迭代器**                               |
| ForwardIterator **max_element**(first, last[, comp]);        | **返回返回中最大元素的迭代器**                               |
| pair<ForwardIterator,ForwardIterator> **minmax_element**(first, last[, comp]); | **返回返回中最小和最大元素的迭代器**                         |

## 7.9 其它

字典序用于获取全排列，字典序从小到大：ABC、ACB、BAC、BCA、CAB、CBA。

| 名称                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| bool **lexicographical_compare**(first1, last1, first2, last2[, comp]); | **字典序的小于比较**<br/>如果范围[first1,last1)的字典序小于范围[first2,last2)则返回true。 |
| bool **next_permutation**(first, last[, comp]);              | **下一个排列**<br/>返回范围的下一个排列，即字典序大1的那个排列。如果能获取更大的字典序排列，返回true。 |
| bool **prev_permutation**(first, last[, comp]);              | **上一个排列**<br/>返回范围的上一个排列，即字典序小1的那个排列。如果能获取更小的字典序排列，返回true。 |

# 8. 其它

## 8.1 bitset

其本质是一个模板类，可以看做是一种类似数组的存储结构。bitset 只能用来存储 bool 类型值，且底层存储机制采用的是用一个比特位来存储一个 bool 值。和 vector 容器不同的是，bitset 的大小在一开始就确定了，因此不支持插入和删除元素；另外 bitset 不是容器，所以不支持使用迭代器。