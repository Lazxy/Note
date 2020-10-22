## 关于LinearLayout中TextView展示不全问题的解释

设想这么一个场景：在一个横向自适应高度LinearLayout布局中，并排放置一个CheckBox和一个TextView，两者都为自适应高度，TextView允许将剩余宽度占满，这是一个典型的协议勾选布局。

一般情况下，这样的布局展示良好，TextView的文字会排列在合适的位置，正好与CheckBox在纵向高度对齐。但是当TextView的文件出现换行时，奇怪的事情就会发生：TextView下方的一部分绘制区域消失了！

在预览窗口中可以看见，TextView的布局范围似乎超出了LinearLayout的绘制范围，并且并没有像预想中的那样从LinearLayout布局的顶部开始布局。这是怎么回事呢？

去LinearLayout的代码中一探究竟，首要怀疑目标当然是onLayout方法中的处理，这里主要是`layoutHorizontal`，在计算子View顶部位置的代码：

```java
childTop = paddingTop + lp.topMargin;
if (childBaseline != -1) {
         childTop += maxAscent[INDEX_TOP] - childBaseline;
 }
```

通过debug可以发现，TextView在布局时，这里的childTop值和CheckBox的0不同，是一个不太大的正数。所以这里的maxAscent和childBaseline分别是怎么来的呢？

一翻代码，前者的值来自于`measureHorizontal`方法：

```java
if (baselineAligned) {
         final int childBaseline = child.getBaseline();
         if (childBaseline != -1) {
                  // Translates the child's vertical gravity into an index in the range 0..2
                 final int gravity = (lp.gravity < 0 ? mGravity : lp.gravity)& Gravity.VERTICAL_GRAVITY_MASK;
                 final int index = ((gravity >> Gravity.AXIS_Y_SHIFT)& ~Gravity.AXIS_SPECIFIED) >> 1;
				maxAscent[index] = Math.max(maxAscent[index], childBaseline);
				maxDescent[index] = Math.max(maxDescent[index],childHeight - childBaseline);
 		}
}
```

这是一个遍历了childView的for循环中的代码，意为根据不同的Gravity值取得该Gravity为该值的View中baseline的最大值。在现在这个例子中，CheckBox的baseline值大于TextView，故maxAscent[INDEX_TOP]的值为CheckBox的baseline值。于是在布局过程中，TextView为了和CheckBox保持`baselineAligned`，即基线对齐，在布局时需要主动向下移动一定范围。

移动位置对齐本来没有什么问题，问题是**在LinearLayout的measure过程中，并没有考虑这种由于基线对齐导致控件部分移出布局范围的情况，故即使此时的高度是wrap_content，但LinearLayout的maxHeight还是只会取子控件中最大的measureHeight，而忽略其为了对齐移动的这些高度**，导致最开头情况的出现。此时可以选择的办法**一是直接将默认为true的baselineAligned参数设为false，二是直接把LinearLayout换成对应的RelativeLayout**。

问题是解决了，但CheckBox和TextView的baseline值究竟是如何定义的呢？此时就需要把目光移向getBaseline方法的实现去了。

虽然看起来不太像，当CheckBox作为Button的隔代子类，实际上最终还是继承自TextView的，故这两者的getBaseline实现实际都在TextView的代码里：

```java
@Override
    public int getBaseline() {
        if (mLayout == null) {
            return super.getBaseline();
        }
        return getBaselineOffset() + mLayout.getLineBaseline(0); 9+23  23 BoringLayout Layout
   }
```

> 在讲到下面的值计算时，需要明确一下基本的定义。在TextView的一行中，可以由上至下画四条横线，分别命名为`top`/`baseline`/`bottom`/`line+1 top`，这是文字的盒模型，其边界范围是上下两行之间，绘制范围是top与bottom之间。
>
> 其中top与baseline的距离称为ascent，baseline与bottom的距离称为descent，bottom与line+1 top的距离为extra，即行距。

对于CheckBox，getBaselineOffset()的值主要来自于其继承自Button的限定高度和文本行高度的差值（即文本行在布局中的下沉高度，这个差值最后也就是和TextView的差值），而TextView的getBaselineOffset()返回值是0，即表示其控件高度与文本行高度相等。虽然结果不同，但是至少其结果的获得方法并没有什么差别。

但对于mLayout.getLineBaseline(0)的实现，两者就不太一样了。CheckBox用的BoringLayout，其目标是一个单行TextView，所以getLineBaseline的值就是`mBottom - mDesc`，标准的定义实现。

TextView的Layout则是更复杂的StaticLayout，其用一个mLine数组存储了各行的各标线的值，这个数组在控件计量结束时就已确定。其内的值依次为：每行的起始字符序号、每行的top值、每行的descent值（这个值会因为断行的小数偏差前后不一）和最后的总高度。





