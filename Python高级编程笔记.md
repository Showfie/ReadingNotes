Chapter 2
---
1 List Comprehensions
```
>>> [i for i in range(10) if i % 2 == 0]
[0, 2, 4, 6, 8]
```
```
>>> def _treatment(pos, element):
...     return '%d: %s' % (pos, element)
...
>>> seq = ["one", "two", "three"]
>>> [_treatment(i, el) for i, el in enumerate(seq)]
['0: one', '1: two', '2: three']
```
2 迭代器和生成器  
* next 返回容器的下一个项目；
* __iter__ 返回迭代器本身；
```
>>> i = iter('abc')
>>> i.next
<method-wrapper 'next' of iterator object at 0x76c16970>
>>> i.next()
'a'
>>> i.next()
'b'
>>> i.next()
'c'
>>> i.next()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  StopIteration
```
generator生成器  
```
>>> def fibonacci():
...     a, b = 0, 1
...     while True:
...             yield b
...             a, b = b, a + b
...
>>> fib = fibonacci()
>>> fib.next()
1
>>> fib.next()
1
>>> fib.next()
2
>>> fib.next()
3
>>> [fib.next() for i in range(10)]
[5, 8, 13, 21, 34, 55, 89, 144, 233, 377]
```

