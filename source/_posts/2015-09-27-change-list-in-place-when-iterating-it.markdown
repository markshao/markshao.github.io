---
layout: post
title: "在迭代过程中修改列表的长度"
date: 2015-09-27 13:31:42 +0800
comments: true
categories: python
---

LeetCode上有一个题目，要求迭代一个列表，然后把列表中所有的0放到整个列表的尾部，保持剩下的元素的顺序

比如说:

输入是  		[1,2,0,3,4,0,5,6,7]
期望的输出是	[1,2,3,4,5,6,7,0,0]

对于这样的一个问题，我们的一个最直接的思路是用一个enumerate来迭代每一个元素，判断是否是0，如果是0的话，就把元素从当前位置pop出来，然后append到尾部

``` python

	for i, v in enumerate(nums):
		if v == 0:
			nums.pop(i)
			nums.append(0)
```

那么这里问题来了，我们一直在迭代的过程中修改列表的长度，而且这里会有一个潜在的问题,一旦我们把某一个元素从列表中pop出来，那么会导致后续元素自动递进一位，这样pop所指向的元素后一位元素，自动被顶到了pop的位置，而解释器认为已经经过了这个位置，也就导致了这个位置的元素被遗漏处理了，我在这里做了一个实验

``` python

	nums = [1,2,3]

	for i,v in enumerate(nums):
		if v == 1:
			nums.pop(i)
			nums.append(4)

		print v
```

我们期待的是打印[1,2,3,4],但是实际的输出结果是[1,3,4],也就是说2这个元素当我们pop(1)的时候，由于自动递进而被忽略了

查看一下python对于enumerate的实现

``` python

	class enumerate(object):
    
    	def __init__(self, iterable, start=0):
        	"""Create an enumerate object.

        	:type iterable: collections.Iterable[T]
        	:type start: numbers.Integral
        	:rtype: enumerate[int, T]
        	"""
        	pass

    	def next(self):
        	"""Return the next value, or raise StopIteration.
	
        	:rtype: (int, T)
        	"""
        	pass

    	def __iter__(self):
        	"""x.__iter__() <==> iter(x).
	
        	:rtype: enumerate[int, T]
        	"""
        	return self
```

enumerate函数主要实现了一个迭代器，最重要的一个元素在这里是start，标示列表中迭代的位置。因此由于列表的顺序发生了变化，但是start还是指向了之前计算出来的结果，所以导致最终读取的元素的错误。

对于以后还有类似于in-place的问题，我们还是要尽可能用循环而不是遍历来解决问题，所以我们把实现最后改成了如下的方式

``` python

	def moveZeroes(nums):
    	"""
    	:type nums: List[int]
    	:rtype: void Do not return anything, modify nums in-place instead.
    	"""
    	length = len(nums)
    	count = i = 0
    	while count < length:
    	    if nums[i] != 0:
    	        i += 1
    	    else:
    	        nums.pop(i)
    	        nums.append(0)
    	    count += 1
```