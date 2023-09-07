# TextView.setText的那些事儿

> 1. 本文基于android-30源码。
> 2. 本文只浅显的梳理下主流程，看源码抓主流程即可，不然会迷失在源码的海洋里。

# 代码

TextView是Android中最基础的组件，也是平时我们使用最多的组件之一。他使用方便，用法简单，简单的`setText` 即可实现功能。那`setText` 到底做了什么呢？

TextView对外提供了几个`setText` 的重载方法，但其实最终都会进入内部的`setText` 方法，代码如下：

```java
private void setText(CharSequence text, BufferType type,
                         boolean notifyBefore, int oldlen) {
	...
		if (mLayout != null) { //1
        checkForRelayout(); //2
	  }
	...
}
```

代码中核心是这一块，有两个需要注意的地方。

1. mLayout ≠ null
    1. mLayout是什么？
        
        mLayout其实就是Layout的子类，并不是我们View绘制中理解的布局，可以理解为一个TextView展示在屏幕上的布局相关的管理信息类。Layout有三个子类：
        
        - BoringLayout
            
            是Layout的最简单的实现，主要用于适配单行文字展示，并且只支持从左到右的展示方向。不建议在自己的开发过程中直接使用，如果需要使用的话，首先使用isBoring判断文字是否符合要求。
            
        - DynamicLayout
            
            支持在排列布局之后修改文字，修改之后会更新text内容。
            
        - StaticLayout
            
            在文字被排列布局之后不允许修改。
            
    2. mLayout什么时候初始化的？
        
        mLayout在`onDraw` 中通过`makeSingleLayout` 初始化。具体逻辑我们下文细说。
        

到了这里，我们知道mLayout肯定不为空，会进入`checkForRelayout` 方法，那我们看看它做了什么。
```java
private void checkForRelayout() {
        //如果width不是wrap_content，则进入该分支（其他条件不重要）
        if ((mLayoutParams.width != LayoutParams.WRAP_CONTENT
                || (mMaxWidthMode == mMinWidthMode && mMaxWidth == mMinWidth))
                && (mHint == null || mHintLayout != null)
                && (mRight - mLeft - getCompoundPaddingLeft() - getCompoundPaddingRight() > 0)) {
                
            int oldht = mLayout.getHeight();
            int want = mLayout.getWidth();
            int hintWant = mHintLayout == null ? 0 : mHintLayout.getWidth();

            //需要关注
            makeNewLayout(want, hintWant, UNKNOWN_BORING, UNKNOWN_BORING,
                          mRight - mLeft - getCompoundPaddingLeft() - getCompoundPaddingRight(),
                          false);

            if (mEllipsize != TextUtils.TruncateAt.MARQUEE) {
                // In a fixed-height view, so use our new text layout.
                if (mLayoutParams.height != LayoutParams.WRAP_CONTENT
                        && mLayoutParams.height != LayoutParams.MATCH_PARENT) {
                    autoSizeText();
                    invalidate();
                    return;
                }

                // Dynamic height, but height has stayed the same,
                // so use our new text layout.
                //如果高度没有变化，则只调用invalidate即可。
                if (mLayout.getHeight() == oldht
                        && (mHintLayout == null || mHintLayout.getHeight() == oldht)) {
                    autoSizeText();
                    invalidate();
                    return;
                }
            }

            // We lose: the height has changed and we have a dynamic height.
            // Request a new view layout using our new text layout.
            requestLayout();
            invalidate();
        } else {
            // Dynamic width, so we have no choice but to request a new
            // view layout with a new text layout.
            //如果是wrap_content则每次需要重新创建layout和整个View视图树重新onLayout
            nullLayouts();
            requestLayout();
            invalidate();
        }
    }
```
上面这段源码的大致逻辑为：
1. 当高度不是WRAP_CONTENT且高度没有变化，则只需要`invalidate`即可；
2. 当高度不是WRAP_CONTENT但高度有变化，则需要重新`requestLayout`和`invalidate`布局；
3. 当高度为WRAP_CONTENT则每次需要置空当前的layout，并且重新`requestLayout`和`invalidate`布局；

还有一个需要我们关注的点是`makeNewLayout`这个方法。
```java
public void makeNewLayout(int wantWidth, int hintWidth,
                                 BoringLayout.Metrics boring,
                                 BoringLayout.Metrics hintBoring,
                                 int ellipsisWidth, boolean bringIntoView) {
        ...
        //mLayout创建
        mLayout = makeSingleLayout(wantWidth, boring, ellipsisWidth, alignment, shouldEllipsize,
                effectiveEllipsize, effectiveEllipsize == mEllipsize);
                
        if (switchEllipsize) {
            TruncateAt oppositeEllipsize = effectiveEllipsize == TruncateAt.MARQUEE
                    ? TruncateAt.END : TruncateAt.MARQUEE;
            mSavedMarqueeModeLayout = makeSingleLayout(wantWidth, boring, ellipsisWidth, alignment,
                    shouldEllipsize, oppositeEllipsize, effectiveEllipsize != mEllipsize);
        }

        shouldEllipsize = mEllipsize != null;
        mHintLayout = null;
        
        //构建用于显示hint的layout
        if (mHint != null) {
            if (shouldEllipsize) hintWidth = wantWidth;

            if (hintBoring == UNKNOWN_BORING) {
                hintBoring = BoringLayout.isBoring(mHint, mTextPaint, mTextDir,
                                                   mHintBoring);
                if (hintBoring != null) {
                    mHintBoring = hintBoring;
                }
            }

            if (hintBoring != null) {
                if (hintBoring.width <= hintWidth
                        && (!shouldEllipsize || hintBoring.width <= ellipsisWidth)) {
                    if (mSavedHintLayout != null) {
                        mHintLayout = mSavedHintLayout.replaceOrMake(mHint, mTextPaint,
                                hintWidth, alignment, mSpacingMult, mSpacingAdd,
                                hintBoring, mIncludePad);
                    } else {
                        mHintLayout = BoringLayout.make(mHint, mTextPaint,
                                hintWidth, alignment, mSpacingMult, mSpacingAdd,
                                hintBoring, mIncludePad);
                    }

                    mSavedHintLayout = (BoringLayout) mHintLayout;
                } else if (shouldEllipsize && hintBoring.width <= hintWidth) {
                    if (mSavedHintLayout != null) {
                        mHintLayout = mSavedHintLayout.replaceOrMake(mHint, mTextPaint,
                                hintWidth, alignment, mSpacingMult, mSpacingAdd,
                                hintBoring, mIncludePad, mEllipsize,
                                ellipsisWidth);
                    } else {
                        mHintLayout = BoringLayout.make(mHint, mTextPaint,
                                hintWidth, alignment, mSpacingMult, mSpacingAdd,
                                hintBoring, mIncludePad, mEllipsize,
                                ellipsisWidth);
                    }
                }
            }
            // TODO: code duplication with makeSingleLayout()
            if (mHintLayout == null) {
                StaticLayout.Builder builder = StaticLayout.Builder.obtain(mHint, 0,
                        mHint.length(), mTextPaint, hintWidth)
                        .setAlignment(alignment)
                        .setTextDirection(mTextDir)
                        .setLineSpacing(mSpacingAdd, mSpacingMult)
                        .setIncludePad(mIncludePad)
                        .setUseLineSpacingFromFallbacks(mUseFallbackLineSpacing)
                        .setBreakStrategy(mBreakStrategy)
                        .setHyphenationFrequency(mHyphenationFrequency)
                        .setJustificationMode(mJustificationMode)
                        .setMaxLines(mMaxMode == LINES ? mMaximum : Integer.MAX_VALUE);
                if (shouldEllipsize) {
                    builder.setEllipsize(mEllipsize)
                            .setEllipsizedWidth(ellipsisWidth);
                }
                mHintLayout = builder.build();
            }
        }

        if (bringIntoView || (testDirChange && oldDir != mLayout.getParagraphDirection(0))) {
            registerForPreDraw();
        }

        if (mEllipsize == TextUtils.TruncateAt.MARQUEE) {
            if (!compressText(ellipsisWidth)) {
                final int height = mLayoutParams.height;
                // If the size of the view does not depend on the size of the text, try to
                // start the marquee immediately
                if (height != LayoutParams.WRAP_CONTENT && height != LayoutParams.MATCH_PARENT) {
                    startMarquee();
                } else {
                    // Defer the start of the marquee until we know our width (see setFrame())
                    mRestartMarquee = true;
                }
            }
        }

        // CursorControllers need a non-null mLayout
        if (mEditor != null) mEditor.prepareCursorControllers();
    }
```
在这个方法中，获取相关的text配置，如对齐方式、省略方式、文字方向等构建出mLayout，根据hint的相关信息构建出mHintLayout。那我们来看看`makeSingleLayout`是怎么配置这个mLayout的。
```java
protected Layout makeSingleLayout(int wantWidth, BoringLayout.Metrics boring, int ellipsisWidth,
            Layout.Alignment alignment, boolean shouldEllipsize, TruncateAt effectiveEllipsize,
            boolean useSaved) {
        Layout result = null;
        //是否创建DynamicLayout
        if (useDynamicLayout()) {
            final DynamicLayout.Builder builder = DynamicLayout.Builder.obtain(mText, mTextPaint,
                    wantWidth)
                    .setDisplayText(mTransformed)
                    .setAlignment(alignment)
                    .setTextDirection(mTextDir)
                    .setLineSpacing(mSpacingAdd, mSpacingMult)
                    .setIncludePad(mIncludePad)
                    .setUseLineSpacingFromFallbacks(mUseFallbackLineSpacing)
                    .setBreakStrategy(mBreakStrategy)
                    .setHyphenationFrequency(mHyphenationFrequency)
                    .setJustificationMode(mJustificationMode)
                    .setEllipsize(getKeyListener() == null ? effectiveEllipsize : null)
                    .setEllipsizedWidth(ellipsisWidth);
            result = builder.build();
        } else {
            //创建BoringLayout
            if (boring == UNKNOWN_BORING) {
                boring = BoringLayout.isBoring(mTransformed, mTextPaint, mTextDir, mBoring);
                if (boring != null) {
                    mBoring = boring;
                }
            }
            if (boring != null) {
                if (boring.width <= wantWidth
                        && (effectiveEllipsize == null || boring.width <= ellipsisWidth)) {
                    //根据相关属性配置，创建对应的Layout，mSavedLayout不为空，则优先使用mSaveLayout创建，如果没有，在创建新的
                    if (useSaved && mSavedLayout != null) {
                        result = mSavedLayout.replaceOrMake(mTransformed, mTextPaint,
                                wantWidth, alignment, mSpacingMult, mSpacingAdd,
                                boring, mIncludePad);
                    } else {
                        result = BoringLayout.make(mTransformed, mTextPaint,
                                wantWidth, alignment, mSpacingMult, mSpacingAdd,
                                boring, mIncludePad);
                    }

                    if (useSaved) {
                        mSavedLayout = (BoringLayout) result;
                    }
                } else if (shouldEllipsize && boring.width <= wantWidth) {
                    if (useSaved && mSavedLayout != null) {
                        result = mSavedLayout.replaceOrMake(mTransformed, mTextPaint,
                                wantWidth, alignment, mSpacingMult, mSpacingAdd,
                                boring, mIncludePad, effectiveEllipsize,
                                ellipsisWidth);
                    } else {
                        result = BoringLayout.make(mTransformed, mTextPaint,
                                wantWidth, alignment, mSpacingMult, mSpacingAdd,
                                boring, mIncludePad, effectiveEllipsize,
                                ellipsisWidth);
                    }
                }
            }
        }
        //兜底逻辑，如果未使用DynamicLayout和BoringLayout，则使用StaticLayout
        if (result == null) {
            StaticLayout.Builder builder = StaticLayout.Builder.obtain(mTransformed,
                    0, mTransformed.length(), mTextPaint, wantWidth)
                    .setAlignment(alignment)
                    .setTextDirection(mTextDir)
                    .setLineSpacing(mSpacingAdd, mSpacingMult)
                    .setIncludePad(mIncludePad)
                    .setUseLineSpacingFromFallbacks(mUseFallbackLineSpacing)
                    .setBreakStrategy(mBreakStrategy)
                    .setHyphenationFrequency(mHyphenationFrequency)
                    .setJustificationMode(mJustificationMode)
                    .setMaxLines(mMaxMode == LINES ? mMaximum : Integer.MAX_VALUE);
            if (shouldEllipsize) {
                builder.setEllipsize(effectiveEllipsize)
                        .setEllipsizedWidth(ellipsisWidth);
            }
            result = builder.build();
        }
        return result;
    }
    
    //是否为spannable且文本可编辑
    public boolean useDynamicLayout() {
        return isTextSelectable() || (mSpannable != null && mPrecomputed == null);
    }
```
这样，一整个完整的setText的Layout创建流程就跟踪完毕了，现在我们回到`checkForResize`方法继续看`makeNewLayout`后的逻辑，假设高度发生了变化，则会先触发`requestLayout`调用TextView的`onLayout`，再通过`invalidate`触发`onDraw`逻辑，一个个来，先看下TextView的`onLayout`逻辑。
```java
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        super.onLayout(changed, left, top, right, bottom);
        if (mDeferScroll >= 0) {
            int curs = mDeferScroll;
            mDeferScroll = -1;
            bringPointIntoView(Math.min(curs, mText.length()));
        }
        // Call auto-size after the width and height have been calculated.
        autoSizeText();
    }

private void autoSizeText() {
        if (!isAutoSizeEnabled()) {
            return;
        }

        if (mNeedsAutoSizeText) {
            if (getMeasuredWidth() <= 0 || getMeasuredHeight() <= 0) {
                return;
            }

            final int availableWidth = mHorizontallyScrolling
                    ? VERY_WIDE
                    : getMeasuredWidth() - getTotalPaddingLeft() - getTotalPaddingRight();
            final int availableHeight = getMeasuredHeight() - getExtendedPaddingBottom()
                    - getExtendedPaddingTop();

            if (availableWidth <= 0 || availableHeight <= 0) {
                return;
            }

            synchronized (TEMP_RECTF) {
                TEMP_RECTF.setEmpty();
                TEMP_RECTF.right = availableWidth;
                TEMP_RECTF.bottom = availableHeight;
                final float optimalTextSize = findLargestTextSizeWhichFits(TEMP_RECTF);

                if (optimalTextSize != getTextSize()) {
                    setTextSizeInternal(TypedValue.COMPLEX_UNIT_PX, optimalTextSize,
                            false /* shouldRequestLayout */);

                    makeNewLayout(availableWidth, 0 /* hintWidth */, UNKNOWN_BORING, UNKNOWN_BORING,
                            mRight - mLeft - getCompoundPaddingLeft() - getCompoundPaddingRight(),
                            false /* bringIntoView */);
                }
            }
        }
        // Always try to auto-size if enabled. Functions that do not want to trigger auto-sizing
        // after the next layout pass should set this to false.
        mNeedsAutoSizeText = true;
    }

```
可以看到`onLayout`大体逻辑就是计算当前字体大小，若有更新，则重新通过`makeNewLayout`创建新的Layout。

> 也可以看出，TextView中所有的Layout创建其实都是通过`makeNewLayout`这个方法来的。

接下来我们看一下`onDraw`方法。
```java

protected void onDraw(Canvas canvas) {
        restartMarqueeIfNeeded();
        // Draw the background for this view
        super.onDraw(canvas);
        
        ...
        if (mLayout == null) {
            assumeLayout();
        }

        Layout layout = mLayout;
        
        canvas.save();
        /*  Would be faster if we didn't have to do this. Can we chop the
            (displayable) text so that we don't need to do this ever?
        */

        int extendedPaddingTop = getExtendedPaddingTop();
        int extendedPaddingBottom = getExtendedPaddingBottom();

        final int vspace = mBottom - mTop - compoundPaddingBottom - compoundPaddingTop;
        final int maxScrollY = mLayout.getHeight() - vspace;

        float clipLeft = compoundPaddingLeft + scrollX;
        float clipTop = (scrollY == 0) ? 0 : extendedPaddingTop + scrollY;
        float clipRight = right - left - getCompoundPaddingRight() + scrollX;
        float clipBottom = bottom - top + scrollY
                - ((scrollY == maxScrollY) ? 0 : extendedPaddingBottom);

        if (mShadowRadius != 0) {
            clipLeft += Math.min(0, mShadowDx - mShadowRadius);
            clipRight += Math.max(0, mShadowDx + mShadowRadius);

            clipTop += Math.min(0, mShadowDy - mShadowRadius);
            clipBottom += Math.max(0, mShadowDy + mShadowRadius);
        }

        canvas.clipRect(clipLeft, clipTop, clipRight, clipBottom);

        int voffsetText = 0;
        int voffsetCursor = 0;

        // translate in by our padding
        /* shortcircuit calling getVerticaOffset() */
        if ((mGravity & Gravity.VERTICAL_GRAVITY_MASK) != Gravity.TOP) {
            voffsetText = getVerticalOffset(false);
            voffsetCursor = getVerticalOffset(true);
        }
        canvas.translate(compoundPaddingLeft, extendedPaddingTop + voffsetText);

        final int layoutDirection = getLayoutDirection();
        final int absoluteGravity = Gravity.getAbsoluteGravity(mGravity, layoutDirection);
        if (isMarqueeFadeEnabled()) {
            if (!mSingleLine && getLineCount() == 1 && canMarquee()
                    && (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) != Gravity.LEFT) {
                final int width = mRight - mLeft;
                final int padding = getCompoundPaddingLeft() + getCompoundPaddingRight();
                final float dx = mLayout.getLineRight(0) - (width - padding);
                canvas.translate(layout.getParagraphDirection(0) * dx, 0.0f);
            }

            if (mMarquee != null && mMarquee.isRunning()) {
                final float dx = -mMarquee.getScroll();
                canvas.translate(layout.getParagraphDirection(0) * dx, 0.0f);
            }
        }

        final int cursorOffsetVertical = voffsetCursor - voffsetText;

        Path highlight = getUpdatedHighlightPath();
        if (mEditor != null) {
            mEditor.onDraw(canvas, layout, highlight, mHighlightPaint, cursorOffsetVertical);
        } else {
            //------
            layout.draw(canvas, highlight, mHighlightPaint, cursorOffsetVertical);
            
        }

        if (mMarquee != null && mMarquee.shouldDrawGhost()) {
            final float dx = mMarquee.getGhostOffset();
            canvas.translate(layout.getParagraphDirection(0) * dx, 0.0f);
            layout.draw(canvas, highlight, mHighlightPaint, cursorOffsetVertical);
        }

        canvas.restore();
    }
```
方法中大体就是根据字体颜色展示顺序，方位关系对canvas进行设置和转化，最终走到`layout.draw`中。
```java
 public void draw(Canvas canvas, Path highlight, Paint highlightPaint,
            int cursorOffsetVertical) {
        final long lineRange = getLineRangeForDraw(canvas); //获取需要绘制的区间行
        int firstLine = TextUtils.unpackRangeStartFromLong(lineRange); //第一行
        int lastLine = TextUtils.unpackRangeEndFromLong(lineRange); //最后一行
        if (lastLine < 0) return;
        //绘制背景
        drawBackground(canvas, highlight, highlightPaint, cursorOffsetVertical,
                firstLine, lastLine);
        //绘制文字
        drawText(canvas, firstLine, lastLine);
    }
    
    public void drawText(Canvas canvas, int firstLine, int lastLine) {
        int previousLineBottom = getLineTop(firstLine);
        int previousLineEnd = getLineStart(firstLine);
        ParagraphStyle[] spans = NO_PARA_SPANS;
        int spanEnd = 0;
        TextPaint paint = mPaint;
        CharSequence buf = mText;

        Alignment paraAlign = mAlignment;
        TabStops tabStops = null;
        boolean tabStopsIsInitialized = false;

        TextLine tl = TextLine.obtain();

        // Draw the lines, one at a time.
        // The baseline is the top of the following line minus the current line's descent.
        for (int lineNum = firstLine; lineNum <= lastLine; lineNum++) {//遍历每行
            int start = previousLineEnd;//开始
            previousLineEnd = getLineStart(lineNum + 1);//记录end
            int end = getLineVisibleEnd(lineNum, start, previousLineEnd);//结束

            int ltop = previousLineBottom;//行top
            int lbottom = getLineTop(lineNum + 1);//行bottom，也就是下一行的top
            previousLineBottom = lbottom;//记录行Bottom
            int lbaseline = lbottom - getLineDescent(lineNum);//行基线，bottom-descent

            int dir = getParagraphDirection(lineNum);//段乱排版方向
            int left = 0;
            int right = mWidth;
　　　　　　　//一：画LeadingMargin
            if (mSpannedText) {//是spannedText
                Spanned sp = (Spanned) buf;//text
                int textLength = buf.length();
                boolean isFirstParaLine = (start == 0 || buf.charAt(start - 1) == '\n');//段落第一行

                // New batch of paragraph styles, collect into spans array.
                // Compute the alignment, last alignment style wins.
                // Reset tabStops, we'll rebuild if we encounter a line with
                // tabs.
                // We expect paragraph spans to be relatively infrequent, use
                // spanEnd so that we can check less frequently.  Since
                // paragraph styles ought to apply to entire paragraphs, we can
                // just collect the ones present at the start of the paragraph.
                // If spanEnd is before the end of the paragraph, that's not
                // our problem.
                if (start >= spanEnd && (lineNum == firstLine || isFirstParaLine)) {
                    spanEnd = sp.nextSpanTransition(start, textLength,
                                                    ParagraphStyle.class);
                    spans = getParagraphSpans(sp, start, spanEnd, ParagraphStyle.class);//获取段落样式

                    paraAlign = mAlignment;//段落对齐方式
                    for (int n = spans.length - 1; n >= 0; n--) {
                        if (spans[n] instanceof AlignmentSpan) {
                            paraAlign = ((AlignmentSpan) spans[n]).getAlignment();
                            break;
                        }
                    }

                    tabStopsIsInitialized = false;
                }
　　　　　　　　　 //画出LeadingMarginSpan
                // Draw all leading margin spans.  Adjust left or right according
                // to the paragraph direction of the line.
                final int length = spans.length;
                boolean useFirstLineMargin = isFirstParaLine;
                for (int n = 0; n < length; n++) {
                    if (spans[n] instanceof LeadingMarginSpan2) {
                        int count = ((LeadingMarginSpan2) spans[n]).getLeadingMarginLineCount();
                        int startLine = getLineForOffset(sp.getSpanStart(spans[n]));
                        // if there is more than one LeadingMarginSpan2, use
                        // the count that is greatest
                        if (lineNum < startLine + count) {
                            useFirstLineMargin = true;
                            break;
                        }
                    }
                }
                for (int n = 0; n < length; n++) {
                    if (spans[n] instanceof LeadingMarginSpan) {//LeadingMarginSpan
                        LeadingMarginSpan margin = (LeadingMarginSpan) spans[n];
                        if (dir == DIR_RIGHT_TO_LEFT) {//右往左
                            margin.drawLeadingMargin(canvas, paint, right, dir, ltop,
                                                     lbaseline, lbottom, buf,
                                                     start, end, isFirstParaLine, this);
                            right -= margin.getLeadingMargin(useFirstLineMargin);
                        } else {//正常阅读顺序
                            margin.drawLeadingMargin(canvas, paint, left, dir, ltop,
                                                     lbaseline, lbottom, buf,
                                                     start, end, isFirstParaLine, this);
                            left += margin.getLeadingMargin(useFirstLineMargin);
                        }
                    }
                }
            }
　　　　　　　//二：Tab或Emoji
            boolean hasTabOrEmoji = getLineContainsTab(lineNum);
            // Can't tell if we have tabs for sure, currently
            if (hasTabOrEmoji && !tabStopsIsInitialized) {
                if (tabStops == null) {
                    tabStops = new TabStops(TAB_INCREMENT, spans);
                } else {
                    tabStops.reset(TAB_INCREMENT, spans);
                }
                tabStopsIsInitialized = true;
            }

            // Determine whether the line aligns to normal, opposite, or center.　　　　　　　//三：对齐方式
            Alignment align = paraAlign;
            if (align == Alignment.ALIGN_LEFT) {
                align = (dir == DIR_LEFT_TO_RIGHT) ?
                    Alignment.ALIGN_NORMAL : Alignment.ALIGN_OPPOSITE;
            } else if (align == Alignment.ALIGN_RIGHT) {
                align = (dir == DIR_LEFT_TO_RIGHT) ?
                    Alignment.ALIGN_OPPOSITE : Alignment.ALIGN_NORMAL;
            }　　　　　　
　　　　　　　//四：获取x轴，然后写字。
            int x;
            if (align == Alignment.ALIGN_NORMAL) {
                if (dir == DIR_LEFT_TO_RIGHT) {
                    x = left + getIndentAdjust(lineNum, Alignment.ALIGN_LEFT);
                } else {
                    x = right + getIndentAdjust(lineNum, Alignment.ALIGN_RIGHT);
                }
            } else {
                int max = (int)getLineExtent(lineNum, tabStops, false);
                if (align == Alignment.ALIGN_OPPOSITE) {
                    if (dir == DIR_LEFT_TO_RIGHT) {
                        x = right - max + getIndentAdjust(lineNum, Alignment.ALIGN_RIGHT);
                    } else {
                        x = left - max + getIndentAdjust(lineNum, Alignment.ALIGN_LEFT);
                    }
                } else { // Alignment.ALIGN_CENTER
                    max = max & ~1;
                    x = ((right + left - max) >> 1) +
                            getIndentAdjust(lineNum, Alignment.ALIGN_CENTER);
                }
            }

            paint.setHyphenEdit(getHyphen(lineNum));
            Directions directions = getLineDirections(lineNum);　　　　　　　　//阅读方式从左向右的，没有tab和emoji表情，非SpannedText，就是最原始最传统最简单画文字cavans.drawText
            if (directions == DIRS_ALL_LEFT_TO_RIGHT && !mSpannedText && !hasTabOrEmoji) {
                // XXX: assumes there's nothing additional to be done
                canvas.drawText(buf, start, end, x, lbaseline, paint);
            } else {//复杂的交给TextLine
                tl.set(paint, buf, start, end, dir, directions, hasTabOrEmoji, tabStops);
                tl.draw(canvas, x, ltop, lbaseline, lbottom);
            }
            paint.setHyphenEdit(0);
        }

        TextLine.recycle(tl);
    }
```
遍历每一行，主要是由四个流程：画LeadingMargin——>确认tab/emoji(TextLine来画)——>根据对齐方式确定从x轴哪个位置开始画(比如居左x就是0咯)——>根据条件判断是交给cavans直接drawText还是TextLine来画字。
好的，接下来看看传统简单文字使用`canvas.drawText`的代码，做了什么。
```java
     /**
     * Draw the specified range of text, specified by start/end, with its origin at (x,y), in the
     * specified Paint. The origin is interpreted based on the Align setting in the Paint.
     *
     * @param text The text to be drawn
     * @param start The index of the first character in text to draw
     * @param end (end - 1) is the index of the last character in text to draw
     * @param x The x-coordinate of origin for where to draw the text
     * @param y The y-coordinate of origin for where to draw the text
     * @param paint The paint used for the text (e.g. color, size, style)
     */
    public void drawText(@NonNull CharSequence text, int start, int end, float x, float y,
            @NonNull Paint paint) {
        super.drawText(text, start, end, x, y, paint);
    }

    public void drawText(@NonNull CharSequence text, int start, int end, float x, float y,
            @NonNull Paint paint) {
        if ((start | end | (end - start) | (text.length() - end)) < 0) {
            throw new IndexOutOfBoundsException();
        }
        throwIfHasHwBitmapInSwMode(paint);
        if (text instanceof String || text instanceof SpannedString ||
                text instanceof SpannableString) {
            nDrawText(mNativeCanvasWrapper, text.toString(), start, end, x, y,
                    paint.mBidiFlags, paint.getNativeInstance());
        } else if (text instanceof GraphicsOperations) {
            ((GraphicsOperations) text).drawText(this, start, end, x, y,
                    paint);
        } else {
            char[] buf = TemporaryBuffer.obtain(end - start);
            TextUtils.getChars(text, start, end, buf, 0);
            nDrawText(mNativeCanvasWrapper, buf, 0, end - start, x, y,
                    paint.mBidiFlags, paint.getNativeInstance());
            TemporaryBuffer.recycle(buf);
        }
    }
```
`canvas.drawText`会直接调用父类的`drawText`，但通过注释说了三个点，一是画多少个字(start-end)，二是从哪开始画，即原点(origin)，这个有xy坐标轴来确定，基于对齐方式的设定，最后就是画笔    paint。说明一下，start和end是从text里面的所以区段，而原点的x轴跟对齐方式相关，y轴一般是baseline。

那如果是比较复杂的内容，则交给TextLine去绘制。
```java
void draw(Canvas c, float x, int top, int y, int bottom) {
        float h = 0;
        final int runCount = mDirections.getRunCount();
        for (int runIndex = 0; runIndex < runCount; runIndex++) {
            final int runStart = mDirections.getRunStart(runIndex);
            if (runStart > mLen) break;
            final int runLimit = Math.min(runStart + mDirections.getRunLength(runIndex), mLen);
            final boolean runIsRtl = mDirections.isRunRtl(runIndex);

            int segStart = runStart;
            for (int j = mHasTabs ? runStart : runLimit; j <= runLimit; j++) {
                if (j == runLimit || charAt(j) == TAB_CHAR) {
                    h += drawRun(c, segStart, j, runIsRtl, x + h, top, y, bottom,
                            runIndex != (runCount - 1) || j != mLen);

                    if (j != runLimit) {  // charAt(j) == TAB_CHAR
                        h = mDir * nextTab(h * mDir);
                    }
                    segStart = j + 1;
                }
            }
        }
    }
```
获取到一些数据配置信息，进入`drawRun`进行行绘制。
```java
private float drawRun(Canvas c, int start,
            int limit, boolean runIsRtl, float x, int top, int y, int bottom,
            boolean needWidth) {

        if ((mDir == Layout.DIR_LEFT_TO_RIGHT) == runIsRtl) {
            float w = -measureRun(start, limit, limit, runIsRtl, null);
            handleRun(start, limit, limit, runIsRtl, c, x + w, top,
                    y, bottom, null, false);
            return w;
        }

        return handleRun(start, limit, limit, runIsRtl, c, x, top,
                y, bottom, null, needWidth);
    }
```
可以看到，最终通过`handleRun`进行绘制。
```java
private float handleRun(int start, int measureLimit,
            int limit, boolean runIsRtl, Canvas c, float x, int top, int y,
            int bottom, FontMetricsInt fmi, boolean needWidth) {

                ...
            x += handleText(activePaint, activeStart, activeEnd, i, inext, runIsRtl, c, x,
                    top, y, bottom, fmi, needWidth || activeEnd < measureLimit,
                    Math.min(activeEnd, mlimit), mDecorations);
                ...
        }

        return x - originalX;
    }
```
这段代码比较长，大体就是判断有误spanned，如果无，则直接`handleText`，如果有则需要对多个不同的MetricAffectingSpan子类进行绘制处理，最后还是会调用`handleText`来进行具体渲染文字操作。

那我们来看看TextLine的`handleText`。
```java
 private float handleText(TextPaint wp, int start, int end,
            int contextStart, int contextEnd, boolean runIsRtl,
            Canvas c, float x, int top, int y, int bottom,
            FontMetricsInt fmi, boolean needWidth, int offset,
            @Nullable ArrayList<DecorationInfo> decorations) {

        if (mIsJustifying) {
            wp.setWordSpacing(mAddedWidthForJustify);
        }
        // Get metrics first (even for empty strings or "0" width runs)
        if (fmi != null) {
            expandMetricsFromPaint(fmi, wp);
        }

        // No need to do anything if the run width is "0"
        if (end == start) {
            return 0f;
        }

        float totalWidth = 0;

        final int numDecorations = decorations == null ? 0 : decorations.size();
        if (needWidth || (c != null && (wp.bgColor != 0 || numDecorations != 0 || runIsRtl))) {
            totalWidth = getRunAdvance(wp, start, end, contextStart, contextEnd, runIsRtl, offset);
        }

        if (c != null) {
            final float leftX, rightX;
            if (runIsRtl) {
                leftX = x - totalWidth;
                rightX = x;
            } else {
                leftX = x;
                rightX = x + totalWidth;
            }
                
                //画背景
            if (wp.bgColor != 0) {
                int previousColor = wp.getColor();
                Paint.Style previousStyle = wp.getStyle();

                wp.setColor(wp.bgColor);
                wp.setStyle(Paint.Style.FILL);
                c.drawRect(leftX, top, rightX, bottom, wp);

                wp.setStyle(previousStyle);
                wp.setColor(previousColor);
            }
            
                //画文字
            drawTextRun(c, wp, start, end, contextStart, contextEnd, runIsRtl,
                    leftX, y + wp.baselineShift);

            if (numDecorations != 0) {
                for (int i = 0; i < numDecorations; i++) {
                    final DecorationInfo info = decorations.get(i);

                    final int decorationStart = Math.max(info.start, start);
                    final int decorationEnd = Math.min(info.end, offset);
                    float decorationStartAdvance = getRunAdvance(
                            wp, start, end, contextStart, contextEnd, runIsRtl, decorationStart);
                    float decorationEndAdvance = getRunAdvance(
                            wp, start, end, contextStart, contextEnd, runIsRtl, decorationEnd);
                    final float decorationXLeft, decorationXRight;
                    if (runIsRtl) {
                        decorationXLeft = rightX - decorationEndAdvance;
                        decorationXRight = rightX - decorationStartAdvance;
                    } else {
                        decorationXLeft = leftX + decorationStartAdvance;
                        decorationXRight = leftX + decorationEndAdvance;
                    }

                    // Theoretically, there could be cases where both Paint's and TextPaint's
                    // setUnderLineText() are called. For backward compatibility, we need to draw
                    // both underlines, the one with custom color first.
                    if (info.underlineColor != 0) {
                        drawStroke(wp, c, info.underlineColor, wp.getUnderlinePosition(),
                                info.underlineThickness, decorationXLeft, decorationXRight, y);
                    }
                    
                    //画下划线
                    if (info.isUnderlineText) {
                        final float thickness =
                                Math.max(wp.getUnderlineThickness(), 1.0f);
                        drawStroke(wp, c, wp.getColor(), wp.getUnderlinePosition(), thickness,
                                decorationXLeft, decorationXRight, y);
                    }
                    
                    //画边框
                    if (info.isStrikeThruText) {
                        final float thickness =
                                Math.max(wp.getStrikeThruThickness(), 1.0f);
                        drawStroke(wp, c, wp.getColor(), wp.getStrikeThruPosition(), thickness,
                                decorationXLeft, decorationXRight, y);
                    }
                }
            }

        }

        return runIsRtl ? -totalWidth : totalWidth;
    }
```
终于到了最终的方法了，也就是TextLine中的`drawTextRun`。
```java

    private void drawTextRun(Canvas c, TextPaint wp, int start, int end,
            int contextStart, int contextEnd, boolean runIsRtl, float x, int y) {

        if (mCharsValid) {
            int count = end - start;
            int contextCount = contextEnd - contextStart;
            c.drawTextRun(mChars, start, count, contextStart, contextCount,
                    x, y, runIsRtl, wp);
        } else {
            int delta = mStart;
            c.drawTextRun(mText, delta + start, delta + end,
                    delta + contextStart, delta + contextEnd, x, y, runIsRtl, wp);
        }
    }
```
所以最终兜兜转转，进行了一系列配置之后，还是回到了`canvas.drawTextRun`来进行文字绘制。

## 大体流程
![setText](setText.png)


## 相关问题

### 1. 缓存相关
其实在Android 4.0 中底层就有引入TextLayoutCache来解决这个问题，每个测量过的文字都被添加到缓存中，下次需要相同的文字时，可以从缓存中获取，不用在测量。不过缓存大小只有0.5 MB。并且在没有缓存之前，我们的首次滑动还是UI线程耗时的。(为了解决这类问题，Android 9.0中添加了PrecomputedText 。据说测量的耗时减少了95%。)
那有缓存就需要有清除缓存的逻辑，那在哪里清楚缓存的呢？上代码。
```java
 public static void freeTextLayoutCaches() {
        nFreeTextLayoutCaches();
    }
```
Canvas中有对应的释放方法，看看哪里有用，有一个统一的对外出口，就是ConfigurationController。
```java
void handleConfigurationChanged(@Nullable Configuration config,
            @Nullable CompatibilityInfo compat) {
            ...
    freeTextLayoutCachesIfNeeded(configDiff);
            ...
    }
```
掐头去尾，是在这个方法中使用了清除缓存的操作，那继续跟踪，这个方法在哪里被调用的？追踪后发现，大致有以下几个地方：
1. ActivityThread.hadnleLaunchActivity
2. ActivityThread.handleRelaunchActivity
3. ActivityThread.handleUpdatePackageCompatibilityInfo
4. ActivityThread.H.CONFIGURATION_CHANGED
5. ActivityThread.handleConfigurationChanged

### 2.TextView导致页面频繁重绘
若TextView频繁修改内容大小，且长度不稳定，则推荐使用MATCH_PARENT或者精确的dp值来设置宽度，不然会引起整个页面的ViewRootImpl的频繁重绘，具体原因见上文中。

## 相关链接
[TextView文字绘制流程](https://www.cnblogs.com/bvin/p/5370490.html)
[TextView使用不当引起的性能问题](https://lqynydyxf.github.io/2020/05/31/TextView%E7%9A%84%E4%B8%8D%E5%BD%93%E4%BD%BF%E7%94%A8%E5%BC%95%E8%B5%B7%E7%9A%84%E6%80%A7%E8%83%BD%E9%97%AE%E9%A2%98/)