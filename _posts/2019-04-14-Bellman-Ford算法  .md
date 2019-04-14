---
layout:     post   				    # 使用的布局（不需要改）
title:      Bellman-Ford单元最短路算法			# 标题 
subtitle:   最短路 #副标题
date:       2019-04-14	# 时间
author:     BY 	KevinHuang					# 作者
header-img: img/post-bg-2019-4-14.jpg 	# 这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - 图
    - 最短路
    - 算法
    
---

# Bellman-Ford算法  
>Bellman-Ford 算法不仅可以求出最短路径，也可以检测负权回路的问题。该算法由美国数学家理查德 • 贝尔曼（Richard Bellman, 动态规划的提出者）和小莱斯特 • 福特（Lester Ford）发明。  


我们在生活中，我们在寻找两个地点最近的路时，一般会有两种考虑：①从现有的路，直达目的②寻找一个中转的地点，使得到目的的距离最短。  

所以此算法也是这样，记从起点s出发到顶点i的距离为d[i]。则：  
```d[i] =  min{d[i], d[j]+w(j,i)}```, ```e(j,i)  属于 E集```。  

这种通过中转点来更新两地之间的最短距离的操作我们称之为”松弛“。如图：  

![](https://ws4.sinaimg.cn/large/006tNc79ly1g226iwpzxlj30ai075glq.jpg)  

i到j的距离为6，但是这时候有个中转点k，可以经过k，使得i到j的最短距离减少为5.这个操作就是松弛。  

我们先来看看Bellman-Ford的核心代码: 

```  
for(int k = 0; k < n; k++)
	for(int i = 0; i < m; i++)
	{
		if(d[v[i]] != INF && d[v[i]] > d[u[i]] + w[i])
			d[v[i]] = d[u[i]] + w[i];	//松弛
	}
``` 

我们为何使用n-1次循环对每条边进行松弛？因为每次循环，我们都可以确定一个点的最短路。当我们第n次循环也进行松弛操作，说明我们的图存在负环！因为我们最短路最多通过n-1条边。  

Bellman-Ford 算法的时间复杂度为 O(nm)，其中 n 为顶点数，m 为边数。
O(nm) 的时间，大多数都浪费了。  

所以有个优化的算法叫SPFA，下次再来谈这个算法😁 


	  

