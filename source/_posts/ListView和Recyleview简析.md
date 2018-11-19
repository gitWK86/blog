---
title: ListView和Recyleview简析
date: 2018-08-29 10:38:16
categories: 
- Android系统
tags:
- Android
- View
---

今天早上是有了第一次电话面试，什么公司就不说了。但毕竟是好久没有参加面试了，有点紧张，问了项目相关的一些问题，自定义View，Http、TCP／IP协议，答的还行吧。有问到ListView的使用，但没问具体的问题，所以就只说了下ListView的优化之类的。刚好想到之前也准备写ListView和Recyleview相关的知识，所以今天就正好写一下，巩固下知识。

## ListView与RecyclerView的区别

#### ListView：

1. Adapter继承的是BaseAdapter 

2. 自定义ViewHolder与convertView的优化（判断是否为null）

3. ListView的分割线直接在布局中设置 divider

4. ListView的点击事件直接是setOnItemClickListener

5. 布局比较单一，只有一个纵向效果

6. 在ListView中通常刷新数据是用notifyDataSetChanged() ，但是这种刷新数据是全局刷新的（每个item的数据都会重新加载一遍），这样一来就会非常消耗资源；


#### RecyclerView：

1. RecyclerView的Adapter继承的是RecyclerView.Adapter

2. RecyclerView的ViewHolder是必须要写的，是强制的，如果不写的话，就不能重写RecyclerView.Adapter中的3个方法 getItemCount()、onCreateViewHolder()、onBindViewHolder()

3. RecyclerView不支持直接在布局中添加分割线

4. RecyclerView不支持点击事件，只能用回调接口来设置点击事件

5. RecyclerView 的布局效果丰富， 可以在LayoutMananger中设置：线性布局（纵向，横向），表格布局，瀑布流布局

6. RecyclerView中可以实现局部刷新，例如：notifyItemChanged()


//未完