## 1.冒泡排序

O(n) = n<sup>2</sup>

冒泡排序通过n-1趟比较，第i趟比较需进行n-i次比较，共需进行n(n-1)/2次比较，
时间复杂度为n<sup>2</sup>。

    :::Python
    def bubble_sort(table):
        length = len(table)
        for i in range(length-1):
            for j in range(length-1-i):
                if table[j] > table[j+1]:
                    table[j], table[j+1] = table[j+1], table[j]

## 2.插入排序

O(n) = n<sup>2</sup>

插入排序在排序第i个数时，前i-1个数已经完成排序，插入后完成前i个数的排序。
最优情况下只需进行n-1次比较，时间复杂度O(n),平均时间复杂度n<sup>2</sup>。

    :::Python
    def insertion_sort(table):
        length = len(table)
        for i in range(length):
            j = i
            current = table[i]
            while j>0 and table[j-1]>current:
                table[j] = table[j-1]
                j -= 1
            table[j] = current

## 3.希尔排序

(最优)O(n) = nlog<sup>2</sup>n

希尔排序基于以下事实：插入排序对已排序表效率较高。按照从大到小不同的步长对表进行
插入排序即实现了希尔排序。根据步长不同时间复杂度不同，目前已知最优步长由[Sedgewick
提出](http://faculty.simpson.edu/lydia.sinapova/www/cmsc250/LN250_Weiss/L12-ShellSort.htm#increments)
： (1, 5, 19, 41, 109,...)

    :::Python
    def shell_sort(table, steps = [109, 41, 19, 5, 1]):
        length = len(table)
        for step in steps:
            for i in range(step, length):
                j = i
                current = table[i]
                while j >= step and table[j-step] > current:
                    table[j] = table[j-step]
                    j -= step
                table[j] = current

## 4.快速排序

(平均)O(n) = nlogn

快速排序首先选择一个项，将小于该项的置于左侧，大于该项的置于右侧，然后分别对
左右两侧进行快速排序。快速排序效率很高。

影响快速排序的一个因素是中枢项的选择。如果选择首个项作为中枢项，那对一个排序
较好的表进行快速排序效率将十分低下，因此选用首、尾、中间三项中间值作为中枢项。

    :::Python
    def quick_sort(table, left=0, right=None):
        if right == None:
            right = len(table) - 1
        center = (left+right) //2
        # 选择中枢项
        if table[left] > table[center]:
            table[left], table[center] = table[center], table[left]
        if table[center] > table[right]:
            table[center], table[right] = table[right], table[center]
        if table[left] > table[center]:
            table[left], table[center] = table[center], table[left]
        # 如果表大小小于3，到此已完成排序
        if right-left <= 2:
            return
        # 将中枢项置于表倒数第二个项
        table[center], table[right-1] = table[right-1], table[center]
        pivot = table[right-1]

        i = left
        j = right - 1

        while True:
            i += 1
            j -= 1
            # 从左开始找到大于中枢项的项
            while table[i] < pivot:
                i += 1
            # 从右开始找到小于中枢项的项
            while table[j] > pivot:
                j -= 1
            # 如果小项位于大项右侧，交换两项
            if i < j:
                table[i], table[j] = table[j], table[i]
            else:
                break
        # 以中枢项为界，左侧均小于中枢项，右侧均大于中枢项。
        table[i], table[right-1] = table[right-1], table[i]
        # 分别对左右区进行快速排序
        quick_sort(table, left, i-1)
        quick_sort(table, i+1, right)

## 5.归并排序

O(n) = nlogn

归并排序需要额外的O(n)空间复杂度。

    :::Python
    def merge_sort(table, left=0, right=None):
        if right == None:
            right = len(table) - 1
        # 单个项直接返回
        if right-left < 1:
            return
        middle = (left+right)//2
        # 先将表分为左右两区，分别对左右两区实现排序。
        merge_sort(table, left, middle)
        merge_sort(table, middle+1, right)
        # 然后合并左右两区
        i = left
        j = middle + 1
        current = left
        sortedTable = []
        while current <= right:
            # 如果右侧表已全部归并完成 或 左侧表最小项小于右侧表最小项，且左侧表还未归并完成
            if j > right or (table[i]<table[j]) and i <= middle:
                sortedTable.append(table[i])
                i += 1
            else:
                sortedTable.append(table[j])
                j += 1
            current += 1
        table[left:right+1] = sortedTable

## 6.堆排序

O(n) = nlogn

利用Python内置模块`heapq`可以快速实现堆排序，不过为了**完整实现**，此处将首先实现堆数据结构，此处默认是优先队列，即根部为最小元素。堆排序需要O(n)的空间复杂度。

### 堆的实现

    :::Python
    class Min:
        '''作为最小根元素'''
        def __lt__(self, other):
            return True
        def __gt__(self, other):
            return False
        def __repr__(self):
            return '-'
    class Heap:
        '''堆的实现'''
        def __init__(self, values=None):
            self.nodes = [Min()]
            if values != None:
                for value in values:
                    self.insert(value)
        @property
        def size(self):
            return len(self.nodes)-1
        def findMin(self):
            return self.nodes[1]
        def insert(self, newValue):
            self.nodes.append(newValue)
            i = self.size
            while self.nodes[i] < self.nodes[i//2]:
                self.nodes[i], self.nodes[i//2] = self.nodes[i//2], self.nodes[i]
                i = i//2
        def deleteMin(self):
            i = 1
            last = self.nodes[self.size]
            while 2*i <= self.size:
                if 2*i+1 <= self.size:
                    smaller = 2*i if self.nodes[2*i] < self.nodes[2*i+1] else 2*i+1
                else:
                    smaller = 2*i
                if self.nodes[smaller] < last:
                    self.nodes[i], self.nodes[smaller] = self.nodes[smaller], self.nodes[i]
                    i = smaller
                else:
                    break
            self.nodes[i], self.nodes[self.size] = last, self.nodes[i]
            return self.nodes.pop()

### 堆排序

    :::Python
    def heap_sort(table):
        heaptable = Heap(table)
        table[:] = [heaptable.deleteMin() for _ in range(heaptable.size)]

## 7.神奇的猴子排序

O(n) = n·n!

既然猴子连莎士比亚的作品都能写出来，排个序应该也不是大问题 :-)。

    :::Python
    import random
    def monkey_sort(table):
        while not all(table[i] <= table[i+1] for i in range(len(table) - 1)):
            random.shuffle(table)

## 8.各排序算法性能比较

测试环境:

- windows 10 专业版 x64
- Intel(R) Core(TM) i7 2.50GHz
- 12GB
- Python3.5.2 x64

分析：

    :::Python
    # 希尔排序使用步长序列[16001, 8929, 3905, 2161, 929, 505, 209, 109, 41, 19, 5, 1]
    sorts = [bubble_sort, insertion_sort, shell_sort, quick_sort, merge_sort, heap_sort]
    def check(n):
        print("# 大小为", n, "的列表排序\n")
        print("# {0:<16}{1:<7}".format("排序算法", "用时"))
        l = [random.randint(0, n) for _ in range(n)]
        for sort in sorts:
            cl = l[:]
            t0 = time.time()
            sort(cl)
            t1 = time.time()
            print("{0:<20}{1:<7.5f}s".format(sort.__name__, t1-t0))
        t2 = time.time()
        sl = sorted(l)
        t3 = time.time()
        print("{0:<20}{1:<7.5f}s".format('tim_sort', t3-t2))

    >>> check(1000)
    # 大小为 1000 的列表排序

    # 排序算法            用时 
    bubble_sort         0.11205s
    insertion_sort      0.08113s
    shell_sort          0.00490s
    quick_sort          0.00781s
    merge_sort          0.02362s
    heap_sort           0.03619s
    tim_sort            0.00097s

    >>> check(10000)
    # 大小为 10000 的列表排序

    # 排序算法            用时 
    bubble_sort         13.63518s
    insertion_sort      8.40691s
    shell_sort          0.05669s
    quick_sort          0.02346s
    merge_sort          0.08884s
    heap_sort           0.16795s
    tim_sort            0.00293s

    # 冒泡与插入排序时间上涨过快，后续将不再分析这两个排序算法

    >>> check(100000)
    # 大小为 100000 的列表排序

    # 排序算法            用时 
    shell_sort          0.80093s
    quick_sort          0.43970s
    merge_sort          0.84578s
    heap_sort           3.01096s
    tim_sort            0.03910s

    >>> check(500000)
    # 大小为 500000 的列表排序

    # 排序算法            用时 
    shell_sort          6.46414s
    quick_sort          2.54591s
    merge_sort          4.98013s
    heap_sort           15.10598s
    tim_sort            0.31891s

可见快速排序在数据量较大时明显优于其他排序算法，而Python内置timsort则明显优于未经优化的快速排序。

---

参考:

+ [Mark Allen Weiss, 《数据结构与算法分析(C语言描述)》](https://book.douban.com/subject/1139426/)

+ [维基百科：希尔排序](https://zh.wikipedia.org/wiki/希尔排序)