---
title: '[算法笔记]使用L形砖拼接国际象棋棋盘'
date: 2017-08-01 19:54:21
tags: [algorithms, Python]
categories: 'note'
---

## 递归解决棋盘拼接问题

### 题目介绍

如图所示，图中有一块角上缺了一个方格的国际象棋棋盘，现在我们想用 L 形砖拼接出这样一块棋盘。

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/chess-board.jpg)

<!--more-->

### Python 代码

```Python

	def cover(board, lab=1, top=0, left=0, side=None):
	  if side is None:
	    side = len(board)

	  s = side // 2

	  offsets = (0, -1), (side-1, 0)

	  for dy_outer, dy_inner in offsets:
	    for dx_outer, dx_inner in offsets:
	      if not board[top+dy_outer][left+dx_outer]:
	        board[top+s+dy_inner][left+s+dx_inner] = lab

	lab += 1
	if s > 1:
	  for dy in [0, s]:
	    for dx in [0, s]:
	      lab = cover(board, lab, top+dy, left+dx, s)

	return lab
```

> 运行一下代码

```Python

	>>> board = [[0]*8 for i in range(8)]
	>>> board[0][7] = -1
	>>> cover(board)
	22
	>>> for row in board:
	...   print((" %2i"*8)%tuple(row))


	  3  3  4  4  8  8  9 -1
	  3  2  2  4  8  7  9  9
	  5  2  6  6 10  7  7 11
	  5  5  6  1 10 10 11 11
	 13 13 14  1  1 18 19 19
	 13 12 14 14 18 18 17 19
	 15 12 12 16 20 17 17 21
	 15 15 16 16 20 20 21 21
```

如上，上面所有的数字标签都排成了 L 形。（-1 除外，那是缺角所在的位置）。这段代码的实现过程基于算法上的归纳法基础和递归的思想。具体解释请参考挪威 Python 程序大拿 Magnus Lie Hetland 写的《Python 算法教程》一书第三章和第六章的知识。
