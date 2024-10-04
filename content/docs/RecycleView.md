---
share: true
---

### stackFromEnd
`stackFromEnd` 是 `RecyclerView` 的一个布局属性，它的作用是改变布局的填充方向。如果设置为 `true`，则布局将从 `RecyclerView` 的底部开始填充，并且第一个项目将在 `RecyclerView` 的底部，这在你想要 `RecyclerView` 显示的内容以底部为基线，或者想要 `RecyclerView` 滚动到底部时非常有用。

例如，如果你正在开发一个聊天应用，你可能希望新的消息出现在屏幕的底部，旧的消息向上滚动。在这种情况下，你可以将 `stackFromEnd` 设置为 `true`。

这是如何设置 `stackFromEnd` 属性的示例：

```java
RecyclerView.LayoutManager layoutManager = new LinearLayoutManager(this);
((LinearLayoutManager) layoutManager).setStackFromEnd(true);
recyclerView.setLayoutManager(layoutManager);
```



> https://blog.csdn.net/HUandroid/article/details/108730940

### 避免刷新闪烁

``` kotlin
// 删除动画保留且不闪烁  
itemAnimator = object : DefaultItemAnimator() {  
    // 会导致闪烁比较严重
    override fun animateAdd(holder: RecyclerView.ViewHolder): Boolean {  
        dispatchAddFinished(holder)  
        return false  
    }  
    // viewholder UI变化的时候会有个透明度、高度的动画
    override fun animateChange(oldHolder: RecyclerView.ViewHolder, newHolder: RecyclerView.ViewHolder,  
	    fromX: Int, fromY: Int, toX: Int, toY: Int): Boolean {  
	    // 不执行animateChange(透明度变化)  
	    dispatchChangeFinished(oldHolder, false)  
	    if (oldHolder !== newHolder) {  
	        dispatchChangeFinished(newHolder, true)  
	    }  
	    return false  
	}
}
```


除了动画导致的闪烁以外，还有因为 notifyItemRangeChanged 导致的多个 item 同时闪烁；
这种情况需要注意设置 setHasStableIds=true，以及 adapter 重写 getItemId，为每个列表项分配一个稳定的 ID，用于优化性能或实现动画效果

#### 这里 post 执行会导致 smooth 效果消失
``` 
public void moveToBottom() {  
    recyclerView.post(() -> {  
        if (linearLayoutManager != null && adapter != null && adapter.getItemCount() > 0) {  
            recyclerView.smoothScrollToPosition(adapter.getItemCount() - 1);  
        }  
    });  
}
```

### ObservableField
使用 ObservableField 替代 viewholder 的 payload 做局部刷新


##  DiffUtil

```kotlin

class PrimaryCommentAdapter(val fragment: VideoFeedCommentDialogFragment) : RecyclerView.Adapter<PrimaryCommentViewHolder>() {  
  
    val data: ArrayList<PrimaryCommentModel> = arrayListOf()  
    private var snapshot: List<Pair<String, String>> = emptyList()  
  
    fun submit(newData: List<PrimaryCommentModel>) {  
        val newSnapshot = newData.map { it.id!! to it.getSnapshotContent() }  
        val differ = DiffUtil.calculateDiff(object : DiffUtil.Callback() {  
            override fun getOldListSize(): Int = snapshot.size  
            override fun getNewListSize(): Int = newSnapshot.size  
            override fun areItemsTheSame(oldItemPosition: Int, newItemPosition: Int): Boolean =  
                snapshot[oldItemPosition].first == newSnapshot[newItemPosition].first  
            override fun areContentsTheSame(oldItemPosition: Int, newItemPosition: Int): Boolean =  
                snapshot[oldItemPosition].second == newSnapshot[newItemPosition].second  
        }, false)  
        data.clear()  
        data.addAll(newData)  
        snapshot = newSnapshot  
        differ.dispatchUpdatesTo(this)  
    }  
  
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): PrimaryCommentViewHolder =  
        PrimaryCommentViewHolder(fragment, VideoFeedPrimaryCommentViewHolderBinding.inflate(LayoutInflater.from(parent.context), parent, false))  
  
    override fun getItemCount(): Int = data.size  
  
    override fun onBindViewHolder(holder: PrimaryCommentViewHolder, position: Int) {  
        holder.onBind(data[position], position)  
    }  
}

---
// model中
fun getSnapshotContent(): String = StringBuilder()  
    .append(has_more).append("_")  
    .append(record?.comment_num ?: 0).append("_")  
    .append(_sendStatus).append("_")  
    .toString()
```


### detectMoves
DiffUtil.calculateDiff (Callback cb, boolean detectMoves) 方法中的 detectMoves 参数用于指示是否需要检测列表中的数据项是否有移动操作。

如果 detectMoves 设置为 true（默认），DiffUtil 将会尝试找出数据项的移动操作，即数据项在旧列表和新列表中的位置发生了变化。这个过程需要额外的计算，但是可以提供更精确的更新结果，尤其是在有动画效果的列表中，可以得到更好的用户体验。

如果 detectMoves 设置为 false，DiffUtil 将不会检测数据项的移动操作，只会检测数据项的添加、删除和修改操作。这样可以减少计算的开销，但是可能会导致更新结果不够精确，尤其是在有动画效果的列表中，可能会影响到用户体验。
![|450](assets/Pasted%20image%2020240423204450.png)

```ad-note

detectMoves为true的时候，如果pos=1的数据换到pos=0，recycleview不会自动滑到新pos=0位置，而是停留在新的pos=1上；原因是diff是从尾部往头部检查数据，所以检查到pos=1和pos=0的时候是直接移动到pos=0前面的，导致原pos=0有可能不在视野内。
detectMoves=false可避免这种情况，当一个数据移动了会先判定为 remove，再判定为 insert。
```

上述动画问题，可能可以通过 setHasStableIds 来规避。


setHasStableIds(boolean hasStableIds)方法用于告知 RecyclerView 每个列表项的 ID 是否固定不变。当 Adapter 的这个设置被激活时（即传入 true），意味着您保证 getItemId(int position)方法返回的每个 ID 在列表中是唯一的并且不会改变。

这个方法的作用主要体现在两个方面：

1. **性能优化**：启用稳定ID可以显著提高RecyclerView的性能。当setHasStableIds(true)被调用时，RecyclerView可以使用这些稳定的ID来避免重复的布局计算和视图重绘，因为它知道即使数据发生变化，每个列表项的ID仍然保持不变。这允许RecyclerView在处理数据集更改时做出更智能的决策，如局部刷新而非全量刷新。
    
2. **改善动画效果**：在数据集发生变化时（如添加、移除、移动等），如果开启了稳定 ID，RecyclerView 可以更准确地识别和定位变化的项，从而产生更平滑的动画效果。RecyclerView 能够利用稳定 ID 追踪哪些项是新的、哪些项被移除，以及哪些项的位置发生了变化，从而为这些变化提供更流畅的视觉反馈。

为了正确使用稳定 ID，需要重写 Adapter 的 getItemId(int position)方法，返回每个项的唯一 ID。




