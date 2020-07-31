---
title:      "RecyclerView之StickyHeader"
date:       2020-07-31
author:     "NOS-AE"
tags:
     - 18级
     - Android
---

*这篇文章主要是介绍一下StikeyHeader我自己的实现方式，至于RecyclerView自身的padding，item的margin等，有需要的话可以自己加上*<br><br>


# 需要实现的核心内容
- 同一个组的标题始终显示在顶部
- 下一个组的标题到达顶部时，把上一个"赖着不走"的标题给顶走

# 利器：ItemDecoration
我使用了ItemDecoration实现粘性头部，可拔插设计，且实现方便<br>
没了解过的小伙伴可以先康康 [参考链接](https://www.jianshu.com/p/b46a4ff7c10a)

# Header分析

先从简单的入手：实现Header，也就是没有Sticky效果，Header会始终跟着列表滑动而滑动<br><br>
首先我们把同一个标题下的Items看作是一个组，那就是要实现组内的第一个Item增加一个Header，此时自然就想到需要重写```getItemOffsets```方法，给需要的Item增加额外空间，然后重写```onDraw```方法绘制Header

# Header实现
```kotlin
class HeaderDecoration(
    private val callback: Callback
): RecyclerView.ItemDecoration() {

    var mHeight = dp2px(30f)

    private val mPaint = Paint().apply {
        color = 0xFFC5C5C5.toInt()
        style = Paint.Style.FILL
        isAntiAlias = true
    }
    private val mTextPaint = TextPaint().apply {
        color = 0xff353535.toInt()
        textSize = sp2dp(14f)
        isAntiAlias = true
        baselineShift = (textSize / 2 - descent()).toInt()
        textAlign = Paint.Align.CENTER
    }

    // 给组内第一个Item增加额外空间来绘制Header
    override fun getItemOffsets(
        outRect: Rect,
        view: View,
        parent: RecyclerView,
        state: RecyclerView.State
    ) {
        super.getItemOffsets(outRect, view, parent, state)
        val pos = parent.getChildAdapterPosition(view)
        if (isFirstInGroup(pos)) {
            outRect.top = mHeight
        }
    }

    // 绘制Header
    override fun onDraw(c: Canvas, parent: RecyclerView, state: RecyclerView.State) {
        super.onDraw(c, parent, state)

        val left = 0f
        val right = parent.width.toFloat()
        val childCount = parent.childCount
        
        for (i in 0 until childCount) {
            val child = parent.getChildAt(i)
            val pos = parent.getChildAdapterPosition(child)
            if (pos == RecyclerView.NO_POSITION || !isFirstInGroup(pos)) continue

            val top = child.top - mHeight.toFloat()
            val bottom = top + mHeight.toFloat()
            c.drawRect(left, top, right, bottom, mPaint)

            val title = callback.getGroupTitle(pos)
            c.drawText(title, left, top + mHeight / 2 + mTextPaint.baselineShift, mTextPaint)
        }
    }

    // 判断item是否所在组的内第一个item
    private fun isFirstInGroup(pos: Int): Boolean {
        if (pos < 0)
            return false
        val index = callback.getGroupIndex(pos)
        val preIndex = callback.getGroupIndex(pos - 1)
        return index != preIndex
    }

    // 回调获取item所在组的下标和标题
    interface Callback {
        fun getGroupIndex(pos: Int): Int
        fun getGroupTitle(pos: Int): String
    }
}
```
**代码说明**：
- 代码是kotlin
- TextPaint和Paint在绘制上没有差别！对某些细节有疑问的可以参考一下[我在StackOverflow上的回答](https://stackoverflow.com/a/61435196/11136727)

- 在getItemOffsets中判断当前需要增加空间的Item是否第组内第一个Item，是的话就增加一个mHeight的高度
- 在onDraw中遍历布局中的itemView，判断这个itemView是不是组内第一个Item，是的话就把Header绘制到这个itemView的正上方，标题则通过回调获得<br>

**使用**
```
    rv.addItemDecoration(HeaderDecoration(object : GroupDecoration.Callback {
            override fun getGroupIndex(pos: Int): Int {
                return findParent(pos) // 具体怎么找到GroupIndex自己实现
            }

            override fun getGroupTitle(pos: Int): String {
                return findParentTitle(pos) // 同上，自己实现
            }

        }))
```

# StickyHeader分析
- StickyHeader是绘制在itemView之上的，所以需要在onDrawOver中绘制。
- StickyHeader要么一直处于顶部不动，要么跟着组内最后一个Item滑走

由此得出实现思路：判断当前RecyclerView最顶部可见的item是不是在组中的最后一个item，是的话，StickyHeader的bottom就要与这个itemView的bottom重合，视觉上就是跟着这个itemView滑走，否则直接在顶部把StickyHeader绘制出来<br>

另外我在实现StickyHeader时发现，之前onDraw绘制的Header，在视觉上不会影响StickyHeader的显示，并且还能巧妙地和onDrawOver绘制的StickyHeader结合形成"下面标题把当前标题顶上去"的效果，所以onDraw的代码不用动，主要添加onDrawOver的实现即可<br><br>

# StickyHeader实现
```kotlin
// 绘制StickyHeader
override fun onDrawOver(c: Canvas, parent: RecyclerView, state: RecyclerView.State) {
    super.onDrawOver(c, parent, state)
    val manager = parent.layoutManager as LinearLayoutManager
    // 找到RecyclerView第一个可见的itemView在adapter中的位置
    val firstPos = manager.findFirstVisibleItemPosition()
    if (firstPos == RecyclerView.NO_POSITION)
        return

    if (isLastInGroup(firstPos)) {
        // 标题要跟着组内最后一个滑走
        val child = parent.findViewHolderForAdapterPosition(firstPos)?.itemView ?: return
        val bottom = min(child.bottom, mHeight).toFloat()
        val top = bottom - mHeight.toFloat()
        val left = 0f
        val right = parent.width.toFloat()
        c.drawRect(left, top, right, bottom, mPaint)

        val title = callback.getGroupTitle(firstPos)
        c.drawText(title, left, top + mHeight / 2 + mTextPaint.baselineShift, mTextPaint)
    } else {
        // 直接绘制在顶部
        val top = 0f
        val bottom = mHeight.toFloat()
        val left = 0f
        val right = parent.width.toFloat()
        c.drawRect(left, top, right, bottom, mPaint)

        val title = callback.getGroupTitle(firstPos)
        c.drawText(title, left, top + mHeight / 2 + mTextPaint.baselineShift, mTextPaint)
    }
}

// 判断item是否所在组的内最后一个item
private fun isLastInGroup(pos: Int): Boolean {
    if (pos < 0)
        return false
    val index = callback.getGroupIndex(pos)
    val preIndex = callback.getGroupIndex(pos + 1)
    return index != preIndex
}
```
**代码说明:**
- 如果item是组内最后一个item，需要用min函数确定StickyHeader的位置，保证StickyHeader能正确显示并随itemView滑出屏幕<br>

**使用:**

- 类似上面普通的Header使用


# 结语
以上就是我实现StickyHeader的实现方法

理解了StickyHeader的实现，类似的例如**CoveredHeader**（即下一个header不是顶走上一个header，而是直接覆盖在上面）也能轻松地自己手动实现