---
share: true
---

超过两行时，maxline 和 ellipsize 不生效
使用这个方法处理
``` kotlin
tvH3.viewTreeObserver.addOnGlobalLayoutListener(object : ViewTreeObserver.OnGlobalLayoutListener {  
	override fun onGlobalLayout() {  
		tvH3.viewTreeObserver.removeOnGlobalLayoutListener(this)  
		if (tvH3.lineCount > 2) {  
			val maxLines = 2  
			val ellipsizeText = TextUtils.ellipsize(data.h3Span, tvH3.paint, (tvH3.width * maxLines).toFloat(), TextUtils.TruncateAt.END)  
			tvH3.text = ellipsizeText  
//                        val lineEndIndex = tvH3.layout.getLineEnd(1)  
//                        val span = URLImageSpanStringBuilder().append(data.h3Span)  
//                        val text = span.replace(lineEndIndex - 1, span.length, "...")  
//                        tvH3.text = text  
		}  
	}  
})
```

优化
```kotlin
tvH3.post {  
    /**  
     * 其它方式如判断行数超过最大限制，然后再截断有个弊端：需要先layout才能获取行数，然后再截断重新layout；布局会有个高度变化。  
     * 为避免高度变化导致布局调整有点明显，先在xml中限定TextView maxLine，内部自动截断超过maxLine行的文案  
     * 然后根据内容字数比对显示字数，当显示字数少于内容字数说明超过maxLine了，就把最后一个字符换成...  
     */    data.h3Span?:return@post  
    var showTextLength = 0  
    for (i in 0 until tvH3.lineCount.coerceAtMost(h3MaxLines)) {  
        val lineStart = tvH3.layout.getLineStart(i)  
        val lineEnd = tvH3.layout.getLineEnd(i)  
        val lineLength = lineEnd - lineStart  
        showTextLength += lineLength  
    }  
    val h3ContentLength = data.h3Span!!.length  
    if (h3ContentLength > showTextLength) {  
        val lineEndIndex = tvH3.layout.getLineEnd(h3MaxLines - 1)  
        val span = URLImageSpanStringBuilder().append(data.h3Span)  
        val text = span.replace(lineEndIndex - 1, span.length, "...")  
        tvH3.text = text  
    }  
}
```



#### 富文本无法点击
可能是加了 `LinkMovementMethod.getInstance()` 支持了链接点击，影响到了正常的点击、长按事件

```xml
<TextView  
    android:id="@+id/tv_h3"  
    app:movementMethod="LinkMovementMethod.getInstance()" />
```



