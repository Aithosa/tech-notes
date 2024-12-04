# 不那么常用的 Java tips

## zip

Python 有个`zip()`函数，可以像拉链一样组合两个列表。
`zipped = zip(1,2,3], [4,5,6,7,8])`

对于 Java，实现这个操作可以如下:

```Java
 List<String> subjectArr = Arrays.asList("aa", "bb", "cc");
List<Long> numArr = Arrays.asList(2L, 6L, 4L);

// @Beta
public static <A, B> List<Pair<A, B>> zipGuava(List<A> as, List<B> bs) {
    return Streams.zip(as.stream(), bs.stream(), Pair::new)
            .collect(Collectors.toList());
}

public static <A, B> List<Pair<A, B>> zipJava8(List<A> as, List<B> bs) {
    return IntStream.range(0, Math.min(as.size(), bs.size()))
            .mapToObj(i -> new Pair<>(as.get(i), bs.get(i)))
            .collect(Collectors.toList());
}

public static <A, B> List<Map.Entry<A, B>> zipJava9(List<A> as, List<B> bs) {
    return IntStream.range(0, Math.min(as.size(), bs.size()))
            .mapToObj(i -> Map.entry(as.get(i), bs.get(i)))
            .collect(Collectors.toList());
}
```

## 匹配文件后缀

一般用于校验接口收到的文件名是不是符合要求

```java
^\S+\.(pdf|doc|docx|txt|jpg|png|bmp|msg)$
^[\s\S]*\.(pdf|doc|docx|txt|jpg|png|bmp|msg)$
^\w*\.(pdf|doc|docx|txt|jpg|png|bmp|msg)$
```

参数的解释：

- `+` 重复一次或更多次
- `*` 重复零次或更多次
- `\s` 匹配任意的空白符
- `\S` 匹配任意不是空白符的字符
- `\W` 匹配任意不是字母，数字，下划线，汉字的字符

使用如:`@Pattern(regexp = "^\\S+\\.pdf|\\S+\\.doc|\\S+\\.docx|\\S+\\.txt|\\S+\\.jpg|\\S+\\.png|\\S+\\.bmp|\\S+\\.msg$", message = "文件上传失败，原因：只能上传doc、docx、txt、pdf、jpg、png、bmp、msg格式的文件")`

需要注意的是，这几个表达式用 SDL Regex Fuzzer 检测处理的都比较慢，之前检测时有几率让程序崩溃，最近没出现过这个情况(应该不存在 ReDos)。
