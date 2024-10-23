
自从去年ChatGPT3\.5发布后使用了几次，现在写代码基本上离不开它和它的衍生产品们了。一方面查资料很方便，快速提炼要点总结；另一方面想写什么样的代码一问就能生成出来，功能大差不差，稍微改改就能用，大大减少使用搜索引擎的时间，是新时代高阶版的Ctrl\+C/V。


不过大语言模型归根揭底是靠训练集训练出来的，它给出的代码还是要自己测试一下用起来才放心，比如这次就被它坑了一把。


**注：因种种原因，本文仅测试了一些国内的大语言模型，没有测试ChatGPT。**


# 原始需求


某列表查询功能，入参包含一组起止日期，需要校验起止日期跨度小于等于200天。


# 前端传参现状


将用户选择的起止日期(yyyy\-MM\-dd)转换成"yyyy\-MM\-dd HH:mm:ss"格式的字符串，且起始时间是"yyyy\-MM\-dd 00:00:00"，终止时间是"yyyy\-MM\-dd 23:59:59"。
比如，页面上选择从2024\-09\-01到2024\-09\-30，实际的入参是"2024\-09\-01 00:00:00"和"2024\-09\-30 23:59:59"。而且对于这组参数，时间跨度是30天，也就是说包括首尾的当天。


# 需求简化


为了便于测试，我先把时间跨度的要求改为2天，比如"2024\-09\-01 00:00:00"到"2024\-09\-02 23:59:59"的时间跨度正好是2天。需求可以简化为：



> String sendDateBegin\="2024\-09\-01 00:00:00"
> String sendDateEnd\="2024\-09\-02 23:59:59"
> Java代码判断两个天数是否小于等于2天


kimi给的结果：
![](https://img2024.cnblogs.com/blog/228024/202410/228024-20241023091850265-1212898517.png)


代码提取出来：



```
    public static void main(String[] args) {
        String sendDateBegin = "2024-09-01 00:00:00";
        String sendDateEnd = "2024-09-04 23:59:59";
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        LocalDateTime beginDate = LocalDateTime.parse(sendDateBegin, formatter);
        LocalDateTime endDate = LocalDateTime.parse(sendDateEnd, formatter);

        // 计算时间差
        Duration duration = Duration.between(beginDate, endDate);

        // 获取天数差
        long daysBetween = duration.toDays();

        // 判断天数是否小于等于2
        if (daysBetween <= 2) {
            System.out.println("两个日期之间的天数小于或等于2天。");
        } else {
            System.out.println("两个日期之间的天数超过2天。");
        }
    }
}

```

# 发现问题


使用2024\-09\-01 00:00:00\~2024\-09\-02 23:59:59这组数据，返回结果如下，看上去一切正常：
![](https://img2024.cnblogs.com/blog/228024/202410/228024-20241023092615743-369326209.png)
换一组输入2024\-09\-01 00:00:00\~2024\-09\-03 23:59:59，居然也告诉我小于等于2天，是哪里出现了问题？
![](https://img2024.cnblogs.com/blog/228024/202410/228024-20241023092822594-2053844925.png)


# 分析


debug一下代码，发现duration.toDays()的实际处理方式是：
将两个时间的秒数差，除以一天包含的秒数(86400\)，两个参数都是long型。
![](https://img2024.cnblogs.com/blog/228024/202410/228024-20241023093000929-1909429057.png)
那么在这个例子里，超过2天但不足3天的数据，由于long的除法，会将小数部分抛弃，再与2比较：2\.999≈2，2≤2为true。这就是为什么看似正确的代码实际引入了一个bug。
而且，即使提醒kimi这段代码有bug并告知对应的输入，给出的答案仍然是错的。


同样的问题通义千问给了另一种解法，但是答案仍然是错的：
![](https://img2024.cnblogs.com/blog/228024/202410/228024-20241023102459206-710723386.png)


再看看文心一言，半斤八两：
![](https://img2024.cnblogs.com/blog/228024/202410/228024-20241023111638183-928960216.png)


# 解决


既然按天比较因为精度丢失而有误差，那么把日期转成毫秒比较就不会丢失精度了，使用如下的判断即可。
`Duration.between(beginDate, endDate).toMillis() <= Duration.ofDays(2).toMillis();`


这个例子生动的展示，**写代码不能完全依赖大语言模型，该做的测试还是要做的。**当然，如果你把测试用例的编写工作也交给了大语言模型，或许是能够测出来bug的，挺讽刺的是不是？
![](https://img2024.cnblogs.com/blog/228024/202410/228024-20241023112308287-856681323.png)


 本博客参考[FlowerCloud机场](https://hushicha.org)。转载请注明出处！
