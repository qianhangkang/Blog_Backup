---
title: JDK9中的String底层改变
date: 2019-11-02 14:43:05
tags: ['Java']
---

在JDK9中String底层存储字符的char数组变成了byte数组，WHY?

<!--more-->

> The problem is, that the vast majority of the strings in applications can be expressed by just one byte using ISO-8859-1/Latin-1 as they contain no special characters. If such strings could be represented with just one byte per character, that would mean only half of the memory would be used to store the String’s characters. Of course, that does not mean that the whole memory consumed by a String object is now only 50% of the original. Not all the memory allocated by strings is used to store the characters.
>
> 应用程序中的绝大多数字符串可以使用一个字节表示，通过ISO-8859-1 / Latin-1编码，因为它们不包含特殊字符。 如果这样的字符串只能用每个字符一个字节表示，那意味着只有一半的内存用于存储字符串的字符。 当然，这并不意味着String对象消耗的整个内存现在只占原始内存的50％。 并非所有由字符串分配的内存都用于存储字符。

**each character is two bytes**

**each Chinese is two character(four bytes)**

# String源码

```
/**
 * The value is used for character storage.
 *
 * @implNote This field is trusted by the VM, and is a subject to
 * constant folding if String instance is constant. Overwriting this
 * field after construction will cause problems.
 *
 * Additionally, it is marked with {@link Stable} to trust the contents
 * of the array. No other facility in JDK provides this functionality (yet).
 * {@link Stable} is safe here, because value is never null.
 */
@Stable
private final byte[] value;

/**
 * The identifier of the encoding used to encode the bytes in
 * {@code value}. The supported values in this implementation are
 *
 * LATIN1
 * UTF16
 *
 * @implNote This field is trusted by the VM, and is a subject to
 * constant folding if String instance is constant. Overwriting this
 * field after construction will cause problems.
 */
private final byte coder;
```

用了byte数组，并用coder标识使用的哪种字符集（LATIN1、UTF16）

















> *大部分正义感都是虚伪的 聊胜于无   —————————————————————————沃德彭尤·胡某*

