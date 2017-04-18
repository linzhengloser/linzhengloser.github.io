---
title: 优雅的用RecyclerView实现单选列表
date: 2016-11-24 17:00:00
categories: 
    - RecyclerView
tags:
	- RecyclerView
---
```java
//根据Position 获取当前以选中的ViewHolder
ViewHolder viewholder = recyclerview.findViewHolderForLayoutPositon(mSelectPosition);
if(viewHolder != null){
    //如果上一个被选中ViewHolder在屏幕内 就改变状态
    viewHolder.checkBox.setChecked(false);
}
//无论在不在屏幕内 都要修改数据源中的状态
mDatas.get(mSelectPosition).setSelected(false);
mSelectPosition = position;
mDatas.get(position).setSelected(true);
holder.checkBox.setChecked(true);
```