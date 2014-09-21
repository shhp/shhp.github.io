---
layout: post
title: 算法题：数字翻译
---

###问题描述

给定一个数字串将其翻译成字母组，其中1->a, 2->b,..., 26->z，另外0x（x不等于0）相当于x. 例如输入12259，可以翻译成lyi，abbei，
lbei,avei或者abyi。输入一个数字串，判断其能否翻译成字母组，若能打印所有可能翻译成的字母组。
<!-- more -->

###递归算法

看到这个问题后，很自然地想到可以使用递归来解决。假设输入的数字串长度为n，它能被翻译成的字母组的个数为L(n). 那么可以得到
递推关系式：

    L(n) = L(n-1) + L(n-2)

其中L(n-1)表示除去第一个数字后，剩下的数字串能翻译成的字母组的个数；L(n-2)表示除去前两个数字后，剩下的数字串能翻译成的字母组的个数。
当L(n)==0时，说明数字串不能被翻译。另外，还需要对一些特殊情况进行分析：

1. 第一个数字为0

   此时由于0不能被翻译成任何一个字母，可以将0直接删除，于是L(n) = L(n-1)。

2. 前两个数字合在一起的值大于26

   此时也有L(n) = L(n-1)。

用Java实现的代码如下：

    ArrayList<String> translateRecursion(String digits) {
        ArrayList<String> result = new ArrayList<String>();
        if (digits.length() == 1) {
            int value = Integer.valueOf(digits);
            if (value == 0) {
                return result;
            } else {
                result.add(String.valueOf(LETTERS[value-1]));
                return result;
            }
        } else if (digits.length() == 2) {
            int firstValue = Integer.valueOf(digits.substring(0, 1));
            int secondValue = Integer.valueOf(digits.substring(1));
            int value = Integer.valueOf(digits);
            if (value == 0) {
                return result;
            } else if (firstValue == 0) {
                result.add(String.valueOf(LETTERS[value-1]));
                return result;
            } else if (secondValue == 0) {
                if (firstValue > 2) {
                    return result;
                } else {
                    result.add(String.valueOf(LETTERS[value-1]));
                    return result;
                }
            } else {
                if (value <= 26) {
                    result.add(String.valueOf(LETTERS[value-1]));
                    result.add(""+LETTERS[firstValue-1]+LETTERS[secondValue-1]);
                    return result;
                } else {
                    result.add(""+LETTERS[firstValue-1]+LETTERS[secondValue-1]);
                    return result;
                }
            }
        } else {
            int firstValue = Integer.valueOf(digits.substring(0, 1));
            int value = Integer.valueOf(digits.substring(0, 2));
            ArrayList<String> first = translateRecursion(digits.substring(1));
            ArrayList<String> firstTwo = translateRecursion(digits.substring(2));
            if (firstValue == 0) {
                return first;
            } else {
                if (value <= 26) {
                    for (String s : first)
                        result.add(LETTERS[firstValue-1]+s);
                    for (String s : firstTwo)
                        result.add(LETTERS[value-1]+s);
                    return result;
                } else {
                    for (String s : first)
                        result.add(LETTERS[firstValue-1]+s);
                    return result;
                }
            }
        }
    }

###动态规划

上面的递归算法会重复计算很多子问题，类似用递归计算斐波那契数列，时间复杂度是指数级的。为了不重复计算子问题，可以将已经算过的子问题结果保存下来，因此
动态规划是一个不错的选择。

我们现在用Trans\[i\]\[j\]表示原数字串中第i位到第j位能被翻译成的字母组的个数，于是有递推关系式：

    Trans[1][m] = Trans[1][m-1] + Trans[1][m-2]

最后的结果则是Trans\[1\]\[n\]。这样只需对数字串进行一次线性扫描即可得到结果。

Java代码如下(这段代码是从后往前扫描)：

    ArrayList<String> translateDP(String digits) {
        Helper[][] dp = new Helper[digits.length()][digits.length()];
        dp[digits.length()-1][digits.length()-1] = new Helper();
        dp[digits.length()-1][digits.length()-1].array = new ArrayList<String>();
        if (digits.charAt(digits.length() - 1) != '0')
            dp[digits.length()-1][digits.length()-1].array.add(
                    String.valueOf(LETTERS[Integer.valueOf(digits.substring(digits.length() - 1, digits.length()))-1]));
        for (int i = digits.length()-2; i >= 0; i--) {
            dp[i][digits.length()-1] = new Helper();
            dp[i][digits.length()-1].array = new ArrayList<String>();
            int firstValue = Integer.valueOf(digits.substring(i, i+1));
            int value = Integer.valueOf(digits.substring(i, i+2));
            if (firstValue == 0) {
                for (String s : dp[i+1][digits.length()-1].array)
                    dp[i][digits.length()-1].array.add(s);
            } else if (value <= 26) {
                for (String s : dp[i+1][digits.length()-1].array)
                    dp[i][digits.length()-1].array.add(LETTERS[firstValue-1]+s);
                if (i+2 >= digits.length())
                    dp[i][digits.length()-1].array.add(""+LETTERS[value-1]);
                else
                    for (String s : dp[i+2][digits.length()-1].array)
                        dp[i][digits.length()-1].array.add(LETTERS[value-1]+s);
            } else {
                for (String s : dp[i+1][digits.length()-1].array)
                    dp[i][digits.length()-1].array.add(LETTERS[firstValue-1]+s);
            }
        }
        
        return dp[0][digits.length()-1].array;
    }
    class Helper {
        ArrayList<String> array;
    }

