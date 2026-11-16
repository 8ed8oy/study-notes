## A
### 快捷键
- ctrl + l 清空终端内容，但是不终止该终端
- 使用 ``exit()`` 而非``command+c``终止任务
### 语法
#### trace
```python
from ucb import trace
@trace
def f(x):
	return ___
// 使用trace后显示函数的调用关系
// 这被称为函数的修饰器
```
#### Y组合子
```python
lambda f: (lambda x: x(x))(lambda x: f(lambda y: x(x)(y)))
```
用法
```python
from operator import sub, mul
def make_anonymous_factorial():
"""Return the value of an expression that computes factorial."""
	return (lambda f: (lambda x: x(x))(lambda x: f(lambda y: x(x)(y))))
	   ( lambda f: lambda n: 1 if n == 1 else mul(n, f(sub(n, 1))) )
```
#### 函数记忆化（计算加速）
```python
def memo(f):
	cache = {}
	def memoized(n):
		if n not in cache:
			cache[n] = f(n)
		return cache[n]
	return memoized
```
