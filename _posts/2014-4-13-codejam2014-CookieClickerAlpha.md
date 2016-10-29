---
layout: post
title: Code Jam 2014之Cookie Clicker Alpha
---

**Problem：**

    In this problem, you start with 0 cookies. You gain cookies at a rate of 
    2 cookies per second, by clicking on a giant cookie. Any time you have 
    at least C cookies, you can buy a cookie farm. Every time you buy a 
    cookie farm, it costs you C cookies and gives you an extra F cookies 
    per second.

    Once you have X cookies that you haven't spent on farms, you win! 
    Figure out how long it will take you to win if you use the best 
    possible strategy.

    Example

    Suppose C=500.0, F=4.0 and X=2000.0. Here's how the best 
    possible strategy plays out:

     1. You start with 0 cookies, but producing 2 cookies per second.
     2. After 250 seconds, you will have C=500 cookies and can buy 
        a farm that produces F=4 cookies per second.
     3. After buying the farm, you have 0 cookies, and your total 
        cookie production is 6 cookies per second.
     4. The next farm will cost 500 cookies, which you can buy 
        after about 83.3333333 seconds.
     5. After buying your second farm, you have 0 cookies, and your 
        total cookie production is 10 cookies per second.
     6. Another farm will cost 500 cookies, which you can buy after
        50 seconds.
     7. After buying your third farm, you have 0 cookies, and your 
        total cookie production is 14 cookies per second.
     8. Another farm would cost 500 cookies, but it actually makes 
        sense not to buy it: instead you can just wait until you have 
        X=2000 cookies, which takes about 142.8571429 seconds.
    
    Total time: 250 + 83.3333333 + 50 + 142.8571429 = 526.1904762 
    seconds.
    
    Notice that you get cookies continuously: so 0.1 seconds after 
    the game starts you'll have 0.2 cookies, and π seconds after 
    the game starts you'll have 2π cookies.

**My solution:**

<!-- more -->

假设当前cookie的增加速率为r，此时有两种选择：

    1. 当拥有足够多的cookie后购买一个farm，然后等待cookie增加到X. 
       所需时间为:
           T1 ＝ C/r + X/(r+F)
    2. 直接等待cookie增加到X. 所需时间为:
           T2 = X/r

然后比较T1与T2. 如果T2较小，则直接等待cookie增加到X，游戏结束；否则，
选择策略1，然后重复以上过程直到T1 > T2.

代码如下：

    double currentRate = 2;
    double costTime = 0;
    double buyFarmTime = 0;
    double waitTillEndTime = 0;
    do {
         buyFarmTime = C/currentRate + X/(currentRate+F);
         waitTillEndTime = X/currentRate;
         if (buyFarmTime <= waitTillEndTime) {
             costTime += C/currentRate;
             currentRate += F;
         } else {
             costTime += waitTillEndTime;
         }
    } while(buyFarmTime <= waitTillEndTime);

这里会有一个问题：为什么当T1>T2时，选择策略2是最优的？有没有可能此后一段
时间先购买若干farm后再等待cookie增加到X，此种情况所用时间比T2更短？答案
是否定的。证明如下：

    现在已知T1>T2, 也即 C/r + X/(r+F) > X/r. 化简可得：
         Cr + CF -XF > 0
    假设现在先购买两个farm，然后等待cookie增加到X，所需时间为：
         T3 = C/r + C/(r+F) + X/(r+2F)
    此时有 T3 - T1 = (C-X)/(r+F) + X/(r+2F)
                  = (Cr+2CF-XF)/[(r+F)(r+2F)] > 0
    也就是说购买两个farm花费的时间比购买一个还要多。
    假设现在购买n个farm以及n+1个farm所需时间分别为Tn和Tn+1, 则有：
          Tn+1 - Tn = (Cr+nCF-XF)/[(r+(n-1)F)(r+nF)] > 0
    这也意味着farm买的越多，到最后所需时间也越多，因此策略2的时间是最短的。