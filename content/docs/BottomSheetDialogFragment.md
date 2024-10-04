---
share: true
---

由于使用 BottomSheetDialogFragment 可能会需要监听监听底部表单的状态变化、拖动事件和高度变化等，所以需要用到 BottomSheetBehavior，而要使用 behavior 就要求对应的布局是 CoordinatorLayout。

如果不希望自己的布局额外再加一层 CoordinatorLayout，可以通过 design_bottom_sheet 获取 BottomSheetDialogFragment 父布局中的 CoordinatorLayout
`BottomSheetBehavior.from(dialog!!.findViewById(R.id.design_bottom_sheet))`

``` kotlin

behavior = BottomSheetBehavior.from(dialog!!.findViewById(R.id.design_bottom_sheet)).apply {  
    peekHeight = mPeekHeight  
    addBottomSheetCallback(object : BottomSheetBehavior.BottomSheetCallback() {  
        override fun onStateChanged(bottomSheet: View, newState: Int) {  
            when (newState) {  
                BottomSheetBehavior.STATE_SETTLING -> {  
                    if (mSlideOffset <= -0.28) {  
                        dismissAllowingStateLoss()  
                    }  
                }  
            }  
        }  
        override fun onSlide(bottomSheet: View, slideOffset: Float) {  
            mSlideOffset = slideOffset  
            onSlide?.invoke((mPeekHeight * slideOffset + mPeekHeight).toInt())  
        }  
    })  
}  
binding.commentLayout.layoutParams = FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, mPeekHeight)
```

![](assets/Pasted%20image%2020240326203240.png)