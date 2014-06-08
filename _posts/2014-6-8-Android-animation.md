---
layout: post
title: Android animation开发笔记
---

最近的工作涉及到Android的animation开发，学到了一些之前没有注意到的知识点，作个记录。

**fillEnable、fillBefore和fillAfter**

   Animation的fillBefore和fillAfter属性分别用来控制动画开始之前或者播放完毕时是否将transformation作用到视图上，默认情况下它们的值分别为true和false。
如果在xml文件中将fillAfter设置为true，那么在动画播放完毕后，视图会停留在动画的最后一个状态，这种效果是符合预期的。但是如果将fillBefore设置为false同时
设置了startOffset，就会发现在动画真正开始前视图就已经处于动画的第一个状态，也就意味着fillBefore＝false不起作用。按照官方文档的说明，fillBefore＝false
只有在fillEnable被设置为true的情况下才会起作用。为何会出现这种奇怪的问题？从源代码中可以找到答案。以下是Animation的getTransformation函数中的几行代码：

    final long startOffset = getStartOffset();
    final long duration = mDuration;
    float normalizedTime;
    if (duration != 0) {
        normalizedTime = ((float) (currentTime - (mStartTime + startOffset))) /
                (float) duration;
    } else {
        // time is a step-change with a zero duration
        normalizedTime = currentTime < mStartTime ? 0.0f : 1.0f;
    }

    final boolean expired = normalizedTime >= 1.0f;
    mMore = !expired;

    if (!mFillEnabled) normalizedTime = Math.max(Math.min(normalizedTime, 1.0f), 0.0f);

    if ((normalizedTime >= 0.0f || mFillBefore) && (normalizedTime <= 1.0f || mFillAfter)) {
        if (!mStarted) {
            fireAnimationStart();
            mStarted = true;
            if (USE_CLOSEGUARD) {
                guard.open("cancel or detach or getTransformation");
            }
        }

        if (mFillEnabled) normalizedTime = Math.max(Math.min(normalizedTime, 1.0f), 0.0f);

        if (mCycleFlip) {
            normalizedTime = 1.0f - normalizedTime;
        }

        final float interpolatedTime = mInterpolator.getInterpolation(normalizedTime);
        applyTransformation(interpolatedTime, outTransformation);
    }

从代码中可以看到当设置了startOffset而当前时间还没到动画真正播放时间时，变量normalizedTime是一个负值。接下来分两种情况讨论：

1. 仅仅设置了fillBefore＝false

   此时fillEnable为默认值false，因此

       if (!mFillEnabled) normalizedTime = Math.max(Math.min(normalizedTime, 1.0f), 0.0f);

   这个if条件满足，从而使得normalizedTime被设置为0，进而使得接下来的if条件也能满足，虽然此时fillBefore＝false，但怎奈normalizedTime已被设置成了0.
   而在这个if语句块中调用了函数applyTransformation。

2. 设置了fillBefore＝false同时fillEnable＝true

   这意味着上面的if语句块不会被执行，因此动画播放前normalizedTime始终为负值，那么接下来的if语句块也不会被执行了。

**AnimationSet的repeatCount**

想让一个动画循环播放，一个简单的方法是在xml文件中将动画的repeatCount属性设置为“infinite”。但是这个方法只能应用于单一动画，如< translate >、< scale >等等，
对< set >却无效。目前的解决方法是给Animation设置一个AnimationListener，在onAnimationEnd方法中调用View.startAnimation(animation)重新播放动画。
