## 按题型记录笔记
###  回溯的写法(17 电话号码的数字组合)
纯递归效率低于回溯，因为：
1. 频繁切片扩容与值拷贝，常数开销大，执行效率低。
2. 每层创建并合并临时切片，存在大量短生命周期对象分配，内存开销高。
```go
func backtrack(参数) {
    if 终止条件 {
        收集结果
        return
    }

    for 选择 := range 本层候选列表 {
        if 不满足剪枝条件 {
            continue是
        }
        做选择 (更新状态/追加 path)
        backtrack(下一层参数)
        撤销选择 (恢复状态/弹出 path)
    }
}
```

---
###  快慢指针的写法(148 排序链表)
1. 快慢指针初始需要一前一后
``` go
slow := head
fast := head.Next
```
### 快速选择(215 数组中的第K个最大元素)
核心思想：基于快排。选定pivot后将比pivot小的放一边，比pivot大的放另一边，然后剪枝。每一轮交换完成后pivot在数组中的位置确定
注意点：必须随机化寻找pivot，不然在某些输入下（有序，大量重复等）复杂度会退化为 $O(N^2)$
核心算法如下：
```go
pivotIndex := left + rand.Intn(right - left + 1)
pivot := nums[pivotIndex]
nums[pivotIndex], nums[right] = nums[right], nums[pivotIndex]
i, j := left, right - 1

for {
	for i <= j && nums[i] < pivot {
		i++
	}
	for i <= j && nums[j] > pivot {
		j--
	}
	if i >= j {
		break
	}
	nums[i], nums[j] = nums[j], nums[i]
	i++
	j--
}
nums[i], nums[right] = nums[right], nums[i]
```
### 堆的写法(347 前k个高频元素)
```go
// 1. 定义堆元素与堆类型
type item struct {
	num   int
	count int
}

// MinHeap 需要实现 heap.Interface
type MinHeap []item

//给切片绑定堆所需的 `Len`、Less、Swap、Push、Pop方法
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i].count < h[j].count } // 小顶堆：比较频次
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

// Push 和 Pop 需要使用指针接收者，因为会修改切片长度
func (h *MinHeap) Push(x any) {
	*h = append(*h, x.(item))
}

func (h *MinHeap) Pop() any {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[:n-1]
	return x
}
// 后续使用如下
h := &MinHeap{}
heap.Init(h)
```
### 动态规划：0/1背包问题
在动态规划中，**0/1 背包**的核心约束是：**每个数字只能用一次，但不同计算路径不应该共享消耗状态**。
```go
// 1. 外层遍历物品（每个数字只登场一次）
for _, num := range nums { 
	// 2. 内层倒序遍历容量
    for j := target; j >= num; j-- {    
        dp[j] = dp[j] || dp[j-num]
    }
}
```

### KMP算法
```go
func strStr(haystack string, needle string) int {
	n := len(needle)
	h := len(haystack)
	if n > h {return -1}
	
	kmp := make([]int, n)
	for i, j := 0, 1; j < n; j++ {
		for i > 0 && needle[j] != needle[i] { //注意这里的回退
			i = kmp[i - 1]
		}
		if needle[j] == needle[i] {
			i++
		}
		kmp[j] = i
	}
	
	for i, j := 0, 0; i < h; i++{
		for j > 0 && haystack[i] != needle[j] {
			j = kmp[j - 1]
		}
		if haystack[i] == needle[j] {
			j++
		}
		if j == n {
			return i + 1 - n
		}
	}
	return -1
}
```
## Go常用库与函数
### 1. `slices` 与 `sort`（切片与排序）
Go 1.21 引入泛型 `slices` 后，大幅取代了部分老旧写法，但两者在二分上各有场景：

|**库**|**常用函数**|**典型用法**|**说明**|
|---|---|---|---|
|`slices`|`slices.Sort(s)`|`slices.Sort(nums)`|原地升序排序（支持所有可比较基础类型）|
|`slices`|`slices.SortFunc(s, cmp)`|`slices.SortFunc(intervals, func(a, b []int) int { return a[0] - b[0] })`|自定义排序（返回值 `<0`, `0`, `>0`）|
|`slices`|`slices.Reverse(s)`|`slices.Reverse(nums)`|原地反转切片|
|`slices`|`slices.Max(s)` / `Min`|`maxVal := slices.Max(nums)`|获取切片最值（切片不可为空）|
|`slices`|`slices.BinarySearch(s, target)`|`idx, found := slices.BinarySearch(nums, x)`|泛型二分精确查找（返回下标与是否存在）|
|`sort`|`sort.Search(n, f)`|`idx := sort.Search(n, func(i int) bool { return nums[i] >= target })`|**最强通用二分**，寻找第一个满足 `f(i) == true` 的下标|
### 2. `math` 与内置函数（数学与极值）
Go 1.21 之前求最值需要转 float64，目前优先使用内置函数：
- **内置函数 `max` / `min`**（Go 1.21+ 全局内置，无需 import）：
```
ans = max(ans, cur)
ans = min(ans, cur, other) // 支持传入多个相同类型的整型/浮点型
```
- **绝对值与极值常量**（需 `import "math"`）：
```
diff := math.Abs(float64(a - b)) // 浮点绝对值，整型常自己写 if a < 0 { -a }
maxInt := math.MaxInt32          // 2^31 - 1
minInt := math.MinInt64          // -2^63
inf := math.MaxInt / 2           // 图论/DP 中防止相加溢出的常用极大值
```
### 3. `strings` 与 `bytes`（字符串与处理）
刷字符串、回文、模拟题必备：
```go
ans.Grow(len(s)) // 预分配容量
```

| **函数**                   | **典型用法**                                                      | **说明**                   |
| ------------------------ | ------------------------------------------------------------- | ------------------------ |
| `strings.Builder`        | `var sb strings.Builder; sb.WriteByte(c); sb.String()`        | 高效拼接字符串，避免 `+` 带来的重复内存分配 |
| `strings.Split` / `Join` | `parts := strings.Split(s, "/")` / `strings.Join(parts, "/")` | 字符串分割与按分隔符拼接             |
| `strings.Repeat`         | `s := strings.Repeat("a", 5)`                                 | 构造重复字符/字符串               |
| `strings.TrimSpace`      | `s = strings.TrimSpace(s)`                                    | 去除首尾空白字符                 |
| `strings.Count`          | `cnt := strings.Count(s, "sub")`                              | 统计子串出现次数                 |
### 4. `strconv`（数值与字符串转换）
常用于大数加减、进制转换或字符串模拟：
- **字符串转数字**：
    ```
    val, err := strconv.Atoi(s)
    num, _ := strconv.ParseInt(s, 10, 64) // 指定进制和位宽
    ```    
- **数字转字符串**： 
    ```
    s := strconv.Itoa(123)
    binStr := strconv.FormatInt(int64(num), 2) // 转二进制字符串
    ```
### 5. `container/heap`（堆 / 优先队列）
Go 标准库没有现成的 `PriorityQueue`，需要对自定义类型实现 `heap.Interface`：
```go
import "container/heap"

type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] } // 小顶堆；h[i] > h[j] 为大顶堆
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x any)        { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() any {
	old := *h
	n := len(old)
	x :=leftMax < rightMax {
             old[n-1]
	*h = old[0 : n-1]
	return x
}

// 使用规范：必须通过 heap.Push / heap.Pop 操作
h := &IntHeap{2, 1, 5}
heap.Init(h)
heap.Push(h, 3)
minVal := heap.Pop(h).(int)
```
### 6. 随机数
rand.Intn(n)或者现代化用法：
```go
import "math/rand/v2"

idx := rand.N(len(this.nums)) // 直接返回 [0, len) 范围的 int
```
## Go 相关问题
#### 1. `++`是语句
下面的写法错误，因为index++被视为语句
```go
backTrack(index++, digits, comb)
```
正确写法是
```go
backTrack(index++, digits, comb)
```

#### 2. 使用位运算快速乘除
```go
// int mid = left + (right - left) / 2 正常写即可
// 现代编译器会自动将上式转化为位运算（left和right均大于零）
middle = ((right - left) >> 1) + left //取中点

i := i << 2 //等价于 i * 4
```

#### 3.切片复制
```go
ans = append(ans, append([]string(nil), plan...))
```
不能直接append，直接append是复制指针
## C专题强化

### 一、刷题必备最小语法

#### **结构与头文件**

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <unordered_set>
#include <unordered_map>
#include <stack>
#include <algorithm> // 包含 sort, max, min 等

using namespace std;
```

#### 变量与常数
- 整型：int, long long（处理溢出）
- 最大/最小值： INT_MAX, INT_MIN (对应 Java 的 Integer.MAX_VALUE)。
- 数组： 永远优先使用 vector\<int> nums。

#### **引用传递**

在递归或辅助函数中，如果不加 &，C++ 会拷贝整个容器，导致超时或逻辑错误。

```cpp
// 错误：path 会被拷贝，修改不影响原件
void dfs(vector<int> path) 

// 正确：引用传递，直接修改原件
void dfs(vector<int>& path)
```

### 二、常见容器API

1. Vector

```cpp

vector<int> v;
v.push_back(1);      // Java: .add(1); Python: .append(1)
v.pop_back();        // 弹出末尾
v.size();            // 长度
v.empty();           // 是否为空
sort(v.begin(), v.end()); // 排序
find(v.begin(), v.end(), 1) != v.end(); // 检查是否有元素 1
```

2. UnorderedMap(哈希表)

```cpp
unordered_map<int, string> m;
m[1] = "zju";        // 赋值
if (m.count(1)) {    // 检查 Key 是否存在 (Java: containsKey)
    cout << m[1];
}
for (auto& [key, val] : m) { // 遍历 (C++17 结构化绑定)
    // 直接使用 key, val
}
```

3. String
cpp中字符串可变

```cpp
string s = "abc";
s[0] = 'z';          // 合法！s 变为 "zbc"
s += "def";          // 拼接
s.substr(1, 2);      // 截取：从索引 1 开始长度为 2 的子串 -> "bc"
```

### 三、常用算法模板

1. 深度优先搜索 (DFS / Backtracking)
这是解决排列组合、路径问题的通用模板。

```cpp
void backtrack(vector<int>& nums, vector<int>& path, vector<bool>& used) {
    if (path.size() == nums.size()) {
        // 找到一个结果
        return;
    }
    for (int i = 0; i < nums.size(); ++i) {
        if (used[i]) continue;
        
        path.push_back(nums[i]); // 选择
        used[i] = true;
        
        backtrack(nums, path, used); // 递归
        
        used[i] = false;         // 回溯（撤销选择）
        path.pop_back();
    }
}
```

2. 二分查找

```cpp
int search(vector<int>& nums, int target) {
    int left = 0, right = nums.size() - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2; // 防溢出
        if (nums[mid] == target) return mid;
        else if (nums[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```

3. 层序遍历（BFS）

```cpp
void bfs(TreeNode* root) {
    if (!root) return;
    queue<TreeNode*> q;
    q.push(root);
    
    while (!q.empty()) {
        int size = q.size(); // 当前层的节点数
        for (int i = 0; i < size; ++i) {
            TreeNode* node = q.front(); q.pop();
            // 处理当前节点
            if (node->left) q.push(node->left);
            if (node->right) q.push(node->right);
        }
    }
}
```
4. 创建链表
```cpp
// 1. 创建哑节点（在栈上创建对象更方便，不需要手动 delete）
ListNode dummy(0); 
// 2. 创建移动指针，初始指向起点
ListNode* curr = &dummy;

// 3. 不断延长它
vector<int> nums = {1, 2, 3, 4};
for (int x : nums) {
    curr->next = new ListNode(x); // 在当前节点后面接上新节点
    curr = curr->next;            // 指针向后移动，准备下一次延长
}

// 4. 最终的链表头就是 dummy.next
ListNode* head = dummy.next;

#-----------------进阶写法----------------#

ListNode* head = nullptr;
ListNode** pp = &head; // pp 指向的是“指针的地址”

for (int i = 1; i <= 4; ++i) {
    *pp = new ListNode(i); // 在当前“空位”填入新节点
    pp = &((*pp)->next);   // 把 pp 挪到新节点的 next 指针的地址上
}
// 此时 head 直接就是链表头，且过程非常丝滑
```
### 其他
1. 万能头文件：#include <bits/stdc++.h>

## 小知识
1. 写全局变量可能导致多个测试样例间污染



