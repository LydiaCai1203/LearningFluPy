## REVIEW OF DATASTRUCTURE

[笨方法学算法](https://python-data-structures-and-algorithms.readthedocs.io/zh/latest/)

------------------
### 1. 几大排序算法

#### 排序算法的执行效率从：
    1. 时间复杂度（最好情况下的时间复杂度，最坏情况下的时间复杂度，平均时间复杂度）
    2. 时间复杂度的系数、常数、低阶 这些都是可以忽略掉的(N特别打的前提下)
    3. 求的其实是比较次数和交换以及移动的次数
    4. 排序算法的内存消耗
    5. 排序算法的稳定性

##### 因为实际开发中，要排序的往往是一组对象，排序的往往是对象的某个key进行排序，但是对象还有其它的属性值，稳定排序算法可以保证对象在排序的时候，key相同的对象之间的相对顺序是正确的。

#### 冒泡排序：
    空间复杂度O(1)       # 是一个原地排序算法
    稳定
    最好的时间复杂度O(N)  # 只需要循环一次，但是元素需要交换N-1次，(N-1)+1=N 
    最坏的时间复杂度O(N**2)  # 最坏的情况就是循环N-1次，[1+2+3+...+(N-1)] + (N-1) 约等于 N**2
    平均的时间复杂度O(N)  # O(N**2)

#### 插入排序
    首先有一个有序数组，将要插入的数依此和这个有序数组里面的每一个数做对比，然后比较大小进行插入。
    空间复杂度O(1) 
    稳定
    最好的时间复杂度O(N)
    最坏的时间复杂度O(N**2)
    平均的时间复杂度O(N**2)

**实现：**

```python
def insertion_sort(array):
    """插排
    """
    for index, value in enumerate(array):
        pos = index
        while pos > 0 and value < array[pos-1]:
            array[pos] = array[pos-1]
            pos -= 1
        array[pos] = value
```

#### 选择排序

    每次都会从无序数组中找出最小的，然后放在有序数组的末尾
    空间复杂度O(1)
    不稳定
    最好的时间复杂度O(N**2)
    最坏的时间复杂度O(N**2)
    平均的时间复杂度O(N**2)

**以上三种排序算法适合小规模的排序**

#### 归并排序
    首先有一堆数组，然后一分为二，再分别将两组一分为二，一直分，直到每个小组都只有两个元素，然后再将小组进行归并，
    合成一个有序的长数组。
    稳定
    最好的时间复杂度O(NlogN)
    最坏的时间复杂度O(NlogN)
    平均的时间复杂度O(NlogN)

```python
def merge_sort(array: list) -> list:
    if len(array) == 1:
        return array
    
    mid = int(len(array) / 2)
    left_array = merge_sort(array[:mid])
    right_array = merge_sort(array[mid:])

    array = merge_two_array(left_array, right_array)
    return array


def merge_two_array(left_array: list, right_array: list) -> list:
    """双指针记录位置
    """
    tmp_array = []
    left, right = 0, 0
    while True:
        
        if left == len(left_array):
            tmp_array.extend(right_array[right:])
            break
        
        if right == len(right_array):
            tmp_array.extend(left_array[left:])
            break
            
        if left < len(left_array) and left_array[left] > right_array[right]:
            tmp_array.append(right_array[right])
            right += 1

        if right < len(right_array) and left_array[left] <= right_array[right]:
            tmp_array.append(left_array[left])
            left += 1
    return tmp_array
```

#### 快速排序

[理解哨兵的算法看这里，这个比较好懂](https://blog.csdn.net/kmyhy/article/details/82991482)

    首先有一堆数组，然后N/2取一个值当pivot, 将所有比pivot小的都排到左边，所有比pivot大的都排到右边去，递归
    空间复杂度 O(1)
    不稳定
    最好的时间复杂度O(NlogN)
    最坏的时间复杂度O(N**2)
    平均的时间复杂度O(NlogN)

**工作过程：**

+ 选择基准值 pivot 将数组分为两个子数组：小于基准值的元素排到左边，大于基准值的元素排到右边，这个过程称为 partition
+ 对子数组再分区，排序，直到有序为止

**实现**：

```python
# v1 -- 1. 这个解法没有考虑有重复值的情况 2. 使用了额外的存储空间 3. 每次都要两次遍历数组
# 这个算法时间复杂度最好的情况也要 O(2NlogN)

def quicksort(array: list):
    if len(array) < 2:
	return array
    
    pivot = array[0]
    left_array = [i for i in array if i < pivot]
    right_array = [i for i in array if i > pivot]
    return quicksort(left_array) + [pivot] + quicksort(right_array)
```

```python
# v2 -- 使用双指针 + pivot 的方式，左指针 负责找到比 pivot 更大的，右指针 负责找到比 pivot 更小的
# 谁找到了就停下，没找的继续往左/右挪，直到 l == r, 然后将 pivot 与 l(r) 指向的元素交换
# 递归重复上面步骤即可

def quick_sort(array, begin, end):
    if begin >= end:
        return list
    
    pivot = array[begin]
    left, right = begin, end

    while left < right:
        # 找到右边比 pivot 大的
        while left < right and pivot <= array[right]:
            right -= 1
        array[left] = array[right]

        # 找到左边比 pivot 小的
        while left < right and pivot >= array[left]:
            left += 1
        array[right] = array[left]
    
    array[left] = pivot
    quick_sort(array, begin, left-1)
    quick_sort(array, left+1, end)
```

### 2. 查找算法

#### 二分查找

```markdown
二分查找的不需要花费额外的空间
需要查找的序列是一个有序序列
最好最坏的时间复杂度应该都是 O(logN)
```

```python
def search(nums: List[int], target: int) -> int:
    if not nums:
        return -1

    begin, end = 0, len(nums) - 1
    while begin <= end:                   # 可以处理 [0] 和 [0, 9] 校准 0 的情况
        mid = int((begin + end) / 2)      # 代表只有两个元素的时候取左边的元素
        if target == nums[mid]:
            return mid
        # 右边
        if target > nums[mid]:
            begin = mid + 1

        if target < nums[mid]:
            end = mid - 1
    return -1
```

```python
# 153. 寻找旋转排序数组中的最小值
# 延伸思考题
def findMin(nums: List[int]) -> int:
    """找到旋转点，就能找到最小值

        [7, 1, 2, 3, 4]
    """
    # 只有一个元素，或者没有旋转过的数组，第一个元素就是最小的
    if len(nums) == 1 or nums[0] < nums[-1]:
        return nums[0]

    begin, end = 0, len(nums) - 1
    while begin <= end:

        mid = int((begin + end) / 2)

        # 如果是最小值直接 return
        if nums[mid - 1] > nums[mid]:
            return nums[mid]
        if nums[mid] > nums[mid + 1]:
            return nums[mid + 1]

        # 代表还在未旋转的部分
        if nums[mid] > nums[0]:
            begin = mid + 1

        if nums[mid] < nums[0]:
            end = mid - 1
```

```python
# 34. 在排序数组中查找元素的第一个和最后一个位置
# 延伸思考题
  def searchRange(nums: List[int], target: int) -> List[int]:
          begin = bin_search_min(nums, target)
          end = bin_search_max(nums, target)

          if not nums or nums[end-1] != target:
              return [-1, -1]

          return [begin, end-1]

  def bin_search_max(array: list, target: int) -> int:
      begin, end = 0, len(array)-1
      while begin <= end:
          mid = int((begin + end) / 2)
          if target >= array[mid]:
              begin = mid + 1
          if target < array[mid]:
              end = mid - 1
      return begin

  def bin_search_min(array: list, target: int) -> int:
      begin, end = 0, len(array)-1
      while begin <= end:
          mid = int((begin + end) / 2)
          if target > array[mid]:
              begin = mid + 1
          if target <= array[mid]:
              end = mid - 1
      return begin
```

### 3. 哈希表

#### 哈希表的工作过程

哈希表的出现旨在能够快速查找和删除元素。给每个元素一种 "逻辑下标"，哈希表就是通过一个哈希函数来计算一个元素应该放在数组哪个位置。当然对于某一个特定的元素，哈希函数每次计算的下标必须要一样才可以，且范围不可以超过给定数组的长度。

假如我们有一个数组 T，有 `M=13` 个元素，可以定义一个简单的哈希函数 `h(key) = key % M` , 但是 key=5 和 key=18 会得到一样的位置结果，这就是 **哈希冲突**。

#### 哈希冲突(collision)

使用 **链接法(chaining)** 解决冲突，将 18 挂在 5 的下面，5 有一个指针指向 8。

使用 **开放寻址法(open addressing)** 解决冲突，当一个槽被占用的时候，采用一种方式来寻找下一个可用的槽。根据寻找槽的方式不同，又可以分为以下几种：

+ **线性探查(line probing)**

  ```shell
  # 这里叫做线性探查的意思应该是下面的这个函数是线性函数
  h(k, i) = (h′(k) + i) % m, i=0,1,...,m-1
  ```

+ **二次探查**

  ```shell
  # 当一个槽被占用，以二次方作为偏移量
  h(k, i) = (h′(k) + c1 + c2i2) % m, i=0,1,...,m−1
  ```

+ **双重散列**

  ```shell
  # 重新计算 hash 结果
  h(k,i) = (h1(k) + ih2(k)) % m
  ```

#### CPython 如何解决冲突

不同 cpython 版本实现的探查方式是不同的，直接查看 dictobject.c 即可。

#### 哈希函数

hash 函数的冲突越小越好，要求尽可能地被散列到 m 个槽中的任何一个。

#### 装载因子(load factor)

如果继续往 hash 表里塞东西，就会发生空间不够用的情况，因此就要 **重新开辟空间并重新进行散列了**。

比如，我们上边的例子插入 8 个元素，哈希表总大小是 13，它的 load factor 就是 `8/13≈0.62` 。这时表示 hash 表很快将会不够用，因此通常的当 `load factor > 0.8` 的时候，表示需要开辟空间，并重新散列了。

#### 重哈希(rehashing)

rehashing 就是重新开辟一片空间，搜索 cpython 的 dictobject.c 文件中的 GROWTH_RATE 关键字即可。python3.3 的策略是扩大为已经使用的槽数目的两倍。开辟了新空间以后，会把原来哈希表里不为空槽的数据重新插入新的哈希表，插入方式不变。

#### 哈希表 ADT

```python
class Slot(object):
		"""一个槽会有三种状态
			1. hashmap.unused        从未使用过 或者 冲突过
			2. hashmap.empty         使用过但是被移除了  
		"""
		def __init__(self, key, value):
				self.key = key
				self.value = value
class HashTable(object):
		pass
```

+ 哈希表的 插入 和 删除 的平均时间复杂度都是 O(1)
+ Slot 在 二次探查法中不能直接删除，原因？(待查)

### 4. 字典 dict

python 内置的 dict 就是利用 hash 表实现的。它支持三个最基本的方法：

+ `add(key, value)` 有 key 则更新，否则插入
+ `get(key, default=None)` 获取 key 的值，不存在则返回默认值 None
+ `removw(key)` 软删除一个 key, 标记 slot 状态为 empty

#### 实现 dict ADT

py2 和 py3 中，dict 的 `items()`、`keys()`、`values()` 的返回时不同的，pt2 返回的是列表，py3 返回的是迭代器，更节约内存空间。

#### Hashtable

作为 dict 的 key 必须是可哈希的，也就是说不能是 list 等可变对象。记得以前看过 《流畅的 Python》我们的对象是一个不可变对象，因此对象也是可以当作 key 的。

### 5. 集合 Set

如果让你自己实现一个 Set 呢？一样可以使用 hash table，只要将 value 设置为 1 即可。
