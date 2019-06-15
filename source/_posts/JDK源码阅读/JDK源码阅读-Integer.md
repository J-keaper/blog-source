---
title: JDK源码阅读-Integer
author: Keaper
tags:
  - JAVA
categories: 
date: 2019-06-15 18:22:00
---

# 主要属性
1. 最基本的是表示包装的基本类型的值：
```java
private final int value;
```
2. 与类型相关的常量
```java
// int能表示的最小值
public static final int   MIN_VALUE = 0x80000000;
// int能表示的最大值
public static final int   MAX_VALUE = 0x7fffffff;
//  代表基本类型int的Class类示例
public static final Class<Integer>  TYPE = (Class<Integer>) Class.getPrimitiveClass("int");

//补码形式的比特数
public static final int SIZE = 32;
//补码形式的字节数
public static final int BYTES = SIZE / Byte.SIZE;
```
3. 用来辅助而定义的常量

`digits`常量中存的是所有可以用来表示数字的字符，最大包含36进制。
```java
final static char[] digits = {
	'0' , '1' , '2' , '3' , '4' , '5' ,
	'6' , '7' , '8' , '9' , 'a' , 'b' ,
	'c' , 'd' , 'e' , 'f' , 'g' , 'h' ,
	'i' , 'j' , 'k' , 'l' , 'm' , 'n' ,
	'o' , 'p' , 'q' , 'r' , 's' , 't' ,
	'u' , 'v' , 'w' , 'x' , 'y' , 'z'
};
```
`DigitTens`和`DigitOnes`分别用来获取一个两位数的十位字符表示和个位字符表示。
```java
final static char [] DigitTens = {
	'0', '0', '0', '0', '0', '0', '0', '0', '0', '0',
	'1', '1', '1', '1', '1', '1', '1', '1', '1', '1',
	'2', '2', '2', '2', '2', '2', '2', '2', '2', '2',
	'3', '3', '3', '3', '3', '3', '3', '3', '3', '3',
	'4', '4', '4', '4', '4', '4', '4', '4', '4', '4',
	'5', '5', '5', '5', '5', '5', '5', '5', '5', '5',
	'6', '6', '6', '6', '6', '6', '6', '6', '6', '6',
	'7', '7', '7', '7', '7', '7', '7', '7', '7', '7',
	'8', '8', '8', '8', '8', '8', '8', '8', '8', '8',
	'9', '9', '9', '9', '9', '9', '9', '9', '9', '9',
	} ;

final static char [] DigitOnes = {
	'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
	'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
	'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
	'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
	'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
	'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
	'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
	'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
	'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
	'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
	} ;
```
上面几个数组主要是用来在转化数字为字符串时通过查表快速获得对应的位。

`sizeTable`主要用来快速判断一个数的位数。
```java
final static int [] sizeTable = { 9, 99, 999, 9999, 99999, 999999, 9999999,
			99999999, 999999999, Integer.MAX_VALUE };

// Requires positive x
static int stringSize(int x) {
	for (int i=0; ; i++)
		if (x <= sizeTable[i])
			return i+1;
}
```

# IntegerCache类
```java
private static class IntegerCache {
	static final int low = -128;
	static final int high;
	static final Integer cache[];

	static {
		// high value may be configured by property
		int h = 127;
		String integerCacheHighPropValue =
			sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
		if (integerCacheHighPropValue != null) {
			try {
				int i = parseInt(integerCacheHighPropValue);
				i = Math.max(i, 127);
				// Maximum array size is Integer.MAX_VALUE
				h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
			} catch( NumberFormatException nfe) {
				// If the property cannot be parsed into an int, ignore it.
			}
		}
		high = h;

		cache = new Integer[(high - low) + 1];
		int j = low;
		for(int k = 0; k < cache.length; k++)
			cache[k] = new Integer(j++);

		// range [-128, 127] must be interned (JLS7 5.1.7)
		assert IntegerCache.high >= 127;
	}

	private IntegerCache() {}
}
```
Integer内部有一个`static`内部类，使用数组缓存了一部分值对应的的包装对象，这部分值的默认范围是[`-128`,`127`]，不过缓存大小可以通过`-XX:AutoBoxCacheMax=<size>`参数来控制，该参数可以设置初始化缓存初始化时`high`的值，但是`low`是不变的，固定为`-128`。
缓存中的值在`public static Integer valueOf(int i)`方法中使用。


# 构造方法
```java
public Integer(int value) {
	this.value = value;
}
// 调用parse方法(后面会讲)将字符串按照十进制解析得到int值
public Integer(String s) throws NumberFormatException {
	this.value = parseInt(s, 10);
}
```

# int->Integer 方法
```java
public static Integer valueOf(int i) {
	if (i >= IntegerCache.low && i <= IntegerCache.high)
		return IntegerCache.cache[i + (-IntegerCache.low)];
	return new Integer(i);
}
```
首先从判断是否在初始化好的`IntegerCache`的缓存范围内，如果在直接返回缓存中的值，否则`new`一个返回。

# String->int 方法
以下这些方法主要用来将`String`类型解析为`int`基本类型：
```java
//解析指定进制的字符串转化为int基本类型
public static int parseInt(String s, int radix);
//上面方法的重载方法，radix = 10
public static int parseInt(String s);

//解析指定进制的无符号字符串转化为int基本类型，即不处理负号
public static int parseUnsignedInt(String s, int radix);
//上面方法的重载方法，radix = 10
public static int parseUnsignedInt(String s);
```
## parseInt方法
```java
public static int parseInt(String s, int radix) throws NumberFormatException{
	if (s == null) {
		throw new NumberFormatException("null");
	}
	if (radix < Character.MIN_RADIX) {
		throw new NumberFormatException("radix " + radix + " less than Character.MIN_RADIX");
	}
	if (radix > Character.MAX_RADIX) {
		throw new NumberFormatException("radix " + radix + " greater than Character.MAX_RADIX");
	}
	int result = 0;
	boolean negative = false;
	int i = 0, len = s.length();
	int limit = -Integer.MAX_VALUE;
	int multmin;
	int digit;
	if (len > 0) {
		char firstChar = s.charAt(0);
		if (firstChar < '0') { // Possible leading "+" or "-"
			if (firstChar == '-') {
				negative = true;
				limit = Integer.MIN_VALUE;
			} else if (firstChar != '+')
				throw NumberFormatException.forInputString(s);

			if (len == 1) // Cannot have lone "+" or "-"
				throw NumberFormatException.forInputString(s);
			i++;
		}
		multmin = limit / radix;
		while (i < len) {
			// Accumulating negatively avoids surprises near MAX_VALUE
			digit = Character.digit(s.charAt(i++),radix);
			if (digit < 0) {
				throw NumberFormatException.forInputString(s);
			}
			// 相乘之前判断防止溢出
			if (result < multmin) {
				throw NumberFormatException.forInputString(s);
			}
			result *= radix;
			// 相减之前判断防止溢出
			if (result < limit + digit) {
				throw NumberFormatException.forInputString(s);
			}
			result -= digit;
		}
	} else {
		throw NumberFormatException.forInputString(s);
	}
	return negative ? result : -result;
}
```
主要逻辑：
1. 判断字符串不为空，并且传进来进制参数在2和36之间。
2. 处理首位符号`+`或者`-`。
3. 接下来就是将字符串转化为数字的逻辑。就是我们熟悉的每一位权重乘以权值（也就是进制数）依次相加，如：127转换成十进制 `1*10*10+2*10+7*1=127`。
	- 但是这里的处理方式不同之处在于统一按照负数来计算而不是按照正数来计算，这是为了在计算得到结果后可以直接将符号位加在前面得到结果。而如果采用正数来计算，正数的范围是无法表示`Integer.MIN_VALUE`的相反数的，这样省去了单独处理`Integer.MIN_VALUE`的逻辑，非常妙。
	- 循环中的两次判断都是为了防止溢出，在相乘或者相减之前提前判断以防止相乘之后溢出导致判断大小出错。

## parseUnsignedInt方法
```java
public static int parseUnsignedInt(String s, int radix)
			throws NumberFormatException {
	if (s == null)  {
		throw new NumberFormatException("null");
	}
	int len = s.length();
	if (len > 0) {
		char firstChar = s.charAt(0);
		if (firstChar == '-') {
			throw new
				NumberFormatException(String.format("Illegal leading minus sign " + "on unsigned string %s.", s));
		} else {
			if (len <= 5 || // Integer.MAX_VALUE in Character.MAX_RADIX is 6 digits
				(radix == 10 && len <= 9) ) { // Integer.MAX_VALUE in base 10 is 10 digits
				return parseInt(s, radix);
			} else {
				long ell = Long.parseLong(s, radix);
				if ((ell & 0xffff_ffff_0000_0000L) == 0) {
					return (int) ell;
				} else {
					throw new
						NumberFormatException(String.format("String value %s exceeds " + "range of unsigned int.", s));
				}
			}
		}
	} else {
		throw NumberFormatException.forInputString(s);
	}
}
```
这个方法是将字符串按照无符号数来处理
1. 如果字符串前有`-`,抛出异常。
2. 对于确定在不会超过`int`最大范围内的数（根据位数判断），调用`Integer.parseInt`方法处理。
3. 有可能超过`int`范围的，调用`Long.parseLong`方法解析为`long`基本类型，如果解析后的值不超过`32`位，强制转换为`int`返回，如果超出了`32`位,抛出异常。
>`Long.parseLong`方法的逻辑与`Integer.parseInt`方法逻辑一致。

# String->Integer 方法
以下这几个方法主要用来将`String`类型转化为`Integer`包装类型。
```java
public static Integer valueOf(String s, int radix) throws NumberFormatException {
	return Integer.valueOf(parseInt(s,radix));
}
public static Integer valueOf(String s) throws NumberFormatException {
	return Integer.valueOf(parseInt(s, 10));
}
```
这两个方法首先将`String`类型解析为`int`基本类型，然后调用`valueOf(int i)`返回其包装类型。

# int->String 方法

## toString方法
以下这几个方法主要用来将`int`基本类型转化为`String`类型。
```java
// 处理任意进制(2-36)，遇到十进制会调用下面一个方法
public static String toString(int i, int radix);
// 只处理十进制，针对十进制进行了优化
public static String toString(int i);
```

### toString(int i, int radix)方法
```java
public static String toString(int i, int radix) {
	if (radix < Character.MIN_RADIX || radix > Character.MAX_RADIX)
		radix = 10;
	/* Use the faster version */
	if (radix == 10) {
		return toString(i);
	}
	char buf[] = new char[33];
	boolean negative = (i < 0);
	int charPos = 32;
	if (!negative) {
		i = -i;
	}
	while (i <= -radix) {
		buf[charPos--] = digits[-(i % radix)];
		i = i / radix;
	}
	buf[charPos] = digits[-i];

	if (negative) {
		buf[--charPos] = '-';
	}
	return new String(buf, charPos, (33 - charPos));
}
```
1. 判断进制，不符合条件按照十进制处理，处理十进制调用`toString(int i)`方法，速度更快。
2. 接下来的过程我们也很熟悉。初始化一个`char`数组，然后就是依次取余，得到其对应的`char`字符，倒序填充数组,在第一位处理符号位。
	- 这里同样是统一转化为负数处理来避免`Integer.MIN_MAX`转化为正数时溢出
	- 用提前初始化好的数组`digits`快速获查表得对应的字符

### toString(int i)方法
```java
public static String toString(int i) {
	// 提前排除 Integer.MIN_VALUE 
	if (i == Integer.MIN_VALUE)
		return "-2147483648";
	// 计算所需位数
	int size = (i < 0) ? stringSize(-i) + 1 : stringSize(i);
	char[] buf = new char[size];
	getChars(i, size, buf);
	return new String(buf, true);
}
static void getChars(int i, int index, char[] buf) {
	int q, r;
	int charPos = index;
	char sign = 0;
	if (i < 0) {
		sign = '-';
		i = -i;
	}
	// Generate two digits per iteration
	while (i >= 65536) {
		q = i / 100;
	// really: r = i - (q * 100);
		r = i - ((q << 6) + (q << 5) + (q << 2));
		i = q;
		buf [--charPos] = DigitOnes[r];
		buf [--charPos] = DigitTens[r];
	}

	// Fall thru to fast mode for smaller numbers
	// assert(i <= 65536, i);
	for (;;) {
		q = (i * 52429) >>> (16+3);
		r = i - ((q << 3) + (q << 1));  // r = i-(q*10) ...
		buf [--charPos] = digits [r];
		i = q;
		if (i == 0) break;
	}
	if (sign != 0) {
		buf [--charPos] = sign;
	}
}
```
这个方法和上一个方法思想是一样的，但是用到几个技巧：  
1. `int`的高两个字节和低两个自己分开处理，低两个字节一次处理两位，高两个字节一次处理一位。
2. 取余数时没有用取模运算，而是先做除法，再做乘法。
	- 一次处理两位时，先除以100，再乘以100，用原数相减得到余数。乘以100用位运算(`q * 100` = `((q << 6) + (q << 5) + (q << 2))`)代替。
	- 一次处理一位时，先除以10，再乘以10，用原数相减得到余数。除以10用乘法和位运算(`i / 10` = `(i * 52429) >>> (16+3)`)代替，乘以10用位运算(`q * 10` = `((q << 3) + (q << 1))`)代替。

> 以上位运算替换均是为了加快计算速度。
> 1. `q * 100` = `((q << 6) + (q << 5) + (q << 2))`  
> 2. `q * 10` = `((q << 3) + (q << 1))`  
> 3. `i / 10` = `(i * 52429) >>> (16+3)`  

> 分别解释一下这三个等式为什么成立：  
> 1. `100 = 2^6 + 2^5 + 2^2` -> `q * 100 = q * (2^6) + q * (2^5) + q * (2^2)` -> `(q << 6) + (q << 5) + (q << 2)`  
> 2. `10 = 2^3 + 2^1` -> `q * 10 = q * (2^3) + q * (2^1)` -> `(q << 3) + (q << 1)`  
> 3. `(i * 52429) >>> (16+3)` -> `(i * 52429) / (2^19)` -> `(i * 52429) / (2^19)` -> `(i * 52429) / 524288` , 这个结果在不考虑余数的情况下等于`i / 10`

解释完正确性之后，来思考几个问题：
1. 为什么要用这么难理解的位运算的来代替乘除呢？显然是为了运算效率，几种运算的效率差别分别为：位运算 > 乘法 > 除法。所以用处理高位时用位运算代替乘法，处理低位时用位运算和乘法代替除法，用位运算代替乘法。
2. 为什么要将高位和低位分开运算呢？答案也是计算速度上的考量，计算高两个字节时可以一次计算两位加快效率，而计算低两个字节时可以通过用乘法和移位代替除法（上面第三个等式）来加快计算效率。

## toUnsignedString方法
以下两个方法将将参数中的`i`作为无符号数转换为`String`。例如：  
`Integer.toString(Integer.MIN_VALUE) -> -2147483648`  
`Integer.toUnsignedString(Integer.MIN_VALUE) -> 2147483648`
```java
public static String toUnsignedString(int i, int radix) {
	// 这个地方其实等于 Long.toString(toUnsignedLong(i), radix);
	// 因为  Long.toUnsignedString(long i, int radix) 在 参数 i >= 0 时调用的就是 Long.toString(long i, int radix)
	return Long.toUnsignedString(toUnsignedLong(i), radix);
}
public static String toUnsignedString(int i) {
	return Long.toString(toUnsignedLong(i));
}
public static long toUnsignedLong(int x) {
	return ((long) x) & 0xffffffffL;
}
```
这两个方法首先都将`i`当成无符号数然后转换为`long`型，也就是保留低`32`位，高`32`位填`0`(转换后的`long`型值一定为正数)。然后调用`Long.toString`方法或者`Long.toUnsignedString`方法得到结果。

# 特殊进制的格式化方法
```java
public static String toOctalString(int i) {
	return toUnsignedString0(i, 3);
}
public static String toHexString(int i) {
	return toUnsignedString0(i, 4);
}
public static String toBinaryString(int i) {
	return toUnsignedString0(i, 1);
}
private static String toUnsignedString0(int val, int shift) {
	// assert shift > 0 && shift <=5 : "Illegal shift value";
	int mag = Integer.SIZE - Integer.numberOfLeadingZeros(val);
	int chars = Math.max(((mag + (shift - 1)) / shift), 1);
	char[] buf = new char[chars];

	formatUnsignedInt(val, shift, buf, 0, chars);

	// Use special constructor which takes over "buf".
	return new String(buf, true);
}
static int formatUnsignedInt(int val, int shift, char[] buf, int offset, int len) {
	int charPos = len;
	int radix = 1 << shift;
	int mask = radix - 1;
	do {
		buf[offset + --charPos] = Integer.digits[val & mask];
		val >>>= shift;
	} while (val != 0 && charPos > 0);

	return charPos;
}
```
这几个方法分别是二进制，八进制，十六进制的格式化方法。之所以这这几个进制有单独的方法，是因为他们都是2的幂次，所以可以直接处理其二进制位。例如八进制就是一次处理三位，十六进制就是一次处理四位。
> 注意这里都是按照无符号数来处理的。

`toUnsignedString0()`方法就是通用的处理方法。
1. 首先计算最后的值的长度，这里并没有考虑符号的问题，统一是按照无符号数来处理的。`((mag + (shift - 1)) / shift)`相当于`(mag / shift)`向上取整。`numberOfLeadingZeros`计算的是前导0的个数，后面会说。
2. `formatUnsignedInt`这个方法比较好理解，就是一次处理`shift`位，方法是与`(1 << shift - 1)`相与，然后右移，处理下一个`shift`位。

# 有关位运算的一些方法
> 代码中的一些位运算方法注释中的 `HD`指的是《Hacker's Delight》一书，其中有解释了大量位运算相关的技巧，中文译本《算法心得：高效算法的奥秘》

## highestOneBit | lowestOneBit
```java
public static int highestOneBit(int i) {
	// HD, Figure 3-1
	i |= (i >>  1);
	i |= (i >>  2);
	i |= (i >>  4);
	i |= (i >>  8);
	i |= (i >> 16);
	return i - (i >>> 1);
}
public static int lowestOneBit(int i) {
	// HD, Section 2-1
	return i & -i;
}
```
这两个方法返回都是2的幂（即二进制表示中至多只有一位是1），两个方法分别表示参数`i`的二进制表示中只保留最高位`1`和只保留最低位`1`后的值。  
例如：`14`的二进制表示为`1110`(省略高位`0`)，`highestOneBit(14) = 8`，即`1000`,`lowestOneBit(i) = 2`，即`0010`。
如果将参数`i`看成二进制的话，就是一个返回最高位`1`的权值，一个返回最低位`1`的权值。将`14`表示为`2`的幂的和：`14 = 8 + 4 + 2`, `highestOneBit`可以得到第一个数，`lowestOneBit`可以得到最后一个数。
但是注意两点：
1. 负数的时候，`highestOneBit`始终返回`-2147483648`，因为最高位始终为符号位`1`。
2. 参数为0时，两个方法返回都是`0`。

这两个方法有什么实际用途呢？
1. `highestOneBit`得到的值等于将参数`x`下调为小于等于`x`且与之最接近的`2`的幂

我们来看一下具体是怎么实现的。  
`highestOneBit`的思想是将最高位的`1`一直向右传播，直到其右方全是`1`，然后将其右边的`0`全部‘减掉’。示例：
```java
i		01000000 00000110 11100011 11011100
i |= (i >>  1)	01100000 00000111 11110011 11111110
i |= (i >>  2)	01111000 00000111 11111111 11111111
i |= (i >>  4)	01111111 10000111 11111111 11111111
i |= (i >>  8)	01111111 11111111 11111111 11111111
i |= (i >>  16)	01111111 11111111 11111111 11111111
i - (i >>> 1)	00100000 00000000 00000000 00000000
```
`lowestOneBit`思想是来自于相反数的补码表示，`一个数的相反数的补码表示为原数的补码表示按位取反，再加1`，加`1`之后，最右边的就会进位一直到第一个`0`也就是原数第一个`1`的位置。最后相与，因为前面的位都是相反的，所以都为`0`，而最低位`1`之后都相等。看个例子就明白了。
```java
132416			00000000 00000010 00000101 01000000
-132416			11111111 11111101 11111010 11000000
132416 & -132416 	00000000 00000000 00000000 01000000
```

## numberOfLeadingZeros | numberOfLeadingZeros（前导0和后导0的个数）
```java
public static int numberOfLeadingZeros(int i) {
	// HD, Figure 5-6
	if (i == 0)
		return 32;
	int n = 1;
	if (i >>> 16 == 0) { n += 16; i <<= 16; }
	if (i >>> 24 == 0) { n +=  8; i <<=  8; }
	if (i >>> 28 == 0) { n +=  4; i <<=  4; }
	if (i >>> 30 == 0) { n +=  2; i <<=  2; }
	n -= i >>> 31;
	return n;
}

public static int numberOfTrailingZeros(int i) {
	// HD, Figure 5-14
	int y;
	if (i == 0) return 32;
	int n = 31;
	y = i <<16; if (y != 0) { n = n -16; i = y; }
	y = i << 8; if (y != 0) { n = n - 8; i = y; }
	y = i << 4; if (y != 0) { n = n - 4; i = y; }
	y = i << 2; if (y != 0) { n = n - 2; i = y; }
	return n - ((i << 1) >>> 31);
}
```
这两个方法是用来计算前导`0`的个数和后导`0`的个数。具体实现方式使用了二分的思想。  
`numberOfLeadingZeros`方法中，首先判断前16位(`i >>> 16`)是不是全为`0`，如果不是则判断前`8`位，如果是则计数增加`16`并且左移`16`位之后再判断前`8`位(这个时候实际判断的是原数的`17-24`位)，然后依次判断前`4`位，前`2`位，前`1`位。  
`numberOfTrailingZeros`方法中，不同的是计数初始值为`31`,首先判断后`16`位(`i << 16`)是不是等于`0`，如果是则判断最后`24`位,如果不是则计数减`16`并且左移`16`位后判断最后`24`位(这个时候判断的实际是原数的最后`8`位)，依次判断后`28`,`30`,`31`位。

我们来看下其中的细节：
`numberOfLeadingZeros`为什么最后一步不太一样呢，其实原本的样子应该是这样的：
```java
......
int n = 0;
......
if (i >>> 31 == 0) { n +=  1; i <<=  1; }
return n;
```
可以将最后一个分支转化为`n = n + 1 - (i >>> 31)`，如果将`n`初始化为`1`，加法也可以省去。为了计算效率，真是操碎了心啊。
`numberOfTrailingZeros`最后一步也不太一样，其实原本的样子是这样的：
```java
......
y = i << 1; if(y != 0) { n = n - 1;}
return n;
```
用`i >>> 31`代替了判断是否为`0`和加`1`操作。

## bitCount（计算1的位数）
```java
public static int bitCount(int i) {
	// HD, Figure 5-2
	i = i - ((i >>> 1) & 0x55555555);
	i = (i & 0x33333333) + ((i >>> 2) & 0x33333333);
	i = (i + (i >>> 4)) & 0x0f0f0f0f;
	i = i + (i >>> 8);
	i = i + (i >>> 16);
	return i & 0x3f;
}
```
`bitCount`方法用来判断二进制表示中`1`的个数。  
先来说一下其原理，基本的思想是分治法，首先将相邻的两位相加结果存放到一个宽度为`2`的位段，此时这个两位值表示的就是原先两位中`1`的个数；然后将相邻两个位段相加结存到一个宽度为`4`的位段中，此时这个四位值表示是原先四位中`1`的个数；依次处理`8`位，`16`位，最后得到的结果就是原数中`1`的个数。  
可以用下面这张图来解释：
![图片来自HD一书](https://blog-picture.nos-eastchina1.126.net/picture0012.jpg)

再来说下代码，首先，代码原型是这样：
```java
x = (x & 0x55555555) + ((x >>>  1) & 0x55555555);
x = (x & 0x33333333) + ((x >>>  2) & 0x33333333);
x = (x & 0x0F0F0F0F) + ((x >>>  4) & 0x0F0F0F0F);
x = (x & 0x00FF00FF) + ((x >>>  8) & 0x00FF00FF);
x = (x & 0x0000FFFF) + ((x >>> 16) & 0x0000FFFF);
return x;
```
这个代码是如何变成上面的那个四不像代码的呢？其实进行了一些优化：  
1. 第一步： 这一步有点看起来像黑魔法，将加法操作变为了减法操作。
我们先只看两位操作，原操作相当于`x & 0x01 + ((x >>> 1) & 0x01)`，而这个黑魔法相当于`x - (x >>> 1 & 0x01)`。  
如何证明这两个操作是等同的呢？  
假设两个位从左到右分别为`b0,b1`，也就是原先的值`x`等于`b0 * 2^1 + b1 * 2^0 = 2b0 + b1`，而我们最后想要的值其实是`b0 + b1`，`b0 + b1 = (2b0 + b1) - b0`，就等于`x - (x >>> 1 & 0x01)`。`32`位一起处理就是上面的结果`i - ((i >>> 1) & 0x55555555)`
2. 第二步：原样不变，没有优化。  
3. 第三步：因为求和操作的结果最多为8，不会超过四位，所以可以先相加后位与  
4. 第四第五步：相加结果不会向相邻位段进位，所以可以先相加后位与，但是因为最多只有32个`1`占6位，所以这两步可以的消位操作可以放到最后一起做。

## rotateLeft | rotateRight （循环左移和循环右移）
```java
public static int rotateLeft(int i, int distance) {
	return (i << distance) | (i >>> -distance);
}

public static int rotateRight(int i, int distance) {
	return (i >>> distance) | (i << -distance);
}
```
这两个方法返回一个int值循环左移和循环右移后的值。`distance`参数中除了最低的五位外都会被忽略(相当于与`0x3f`相与)，可以认为范围始终在`0-32`之间，这与`java语言规范`中`<<`,`>>`,`>>>`指令的语义规范一致。
> If the promoted type of the left-hand operand is int, then only the five lowest-order
bits of the right-hand operand are used as the shift distance. It is as if the right-hand
operand were subjected to a bitwise logical AND operator & with the
mask value 0x1f (0b11111). The shift distance actually used is therefore always in
the range 0 to 31, inclusive.  

代码很好理解，左移就是左移`d`位后的值与右移`32-d`后的值相与，就相当于将左移溢出的部分补全到右边的位置；右移同理。那么代码中的`(i >>> -distance)`和`(i << -distance)`是怎么回事呢？其实`-distance & 0x1f`就等于`32 - distance`。所以这里可以直接用`-distance`来做运算。

## reverse（比特翻转）
```java
public static int reverse(int i) {
	// HD, Figure 7-1
	i = (i & 0x55555555) << 1 | (i >>> 1) & 0x55555555;
	i = (i & 0x33333333) << 2 | (i >>> 2) & 0x33333333;
	i = (i & 0x0f0f0f0f) << 4 | (i >>> 4) & 0x0f0f0f0f;
	i = (i << 24) | ((i & 0xff00) << 8) |
		((i >>> 8) & 0xff00) | (i >>> 24);
	return i;
}
```
`reverse`方法将`int`型值的二进制补码表示翻转顺序。思想也比较简单，还是分治思想，首先将以`1bit`为单位翻转顺序然后以`2bit`为单位翻转顺序，依次进行。
为什么最后一句不太一样，因为到了按照`8bit`操作时，直接按照`byte`来操作了。

## reverseBytes（字节翻转）
```java
public static int reverseBytes(int i) {
	return ((i >>> 24)) |
		((i >>   8) &   0xFF00) |
		((i <<   8) & 0xFF0000) |
		((i << 24));
}
```
将`int`型值的二进制补码表示以`byte`为单位翻转。

## signum （计算符号位）
```java
public static int signum(int i) {
	// HD, Section 2-7
	return (i >> 31) | (-i >>> 31);
}
```
`signum`方法用来计算一个`int`型值的符号，正数返回`1`,负数返回`-1`,`0`返回`0`。
分别考虑`|`两边的式子  
`(i >> 31)`在`i>0`时返回`0`,`i<0`时返回`-1`,`i=0`时返回`0`
`(-i >>> 31)`在`i>0`是返回`1`,`i<0`时返回`0`,`i=0`时返回`0`
可见，两个式子分别能满足正数和负数的返回值要求，将两者相与便都可以可以覆盖正数和负数。


# 无符号除法
```java
public static int divideUnsigned(int dividend, int divisor) {
	// In lieu of tricky code, for now just use long arithmetic.
	return (int)(toUnsignedLong(dividend) / toUnsignedLong(divisor));
}

public static int remainderUnsigned(int dividend, int divisor) {
	// In lieu of tricky code, for now just use long arithmetic.
	return (int)(toUnsignedLong(dividend) % toUnsignedLong(divisor));
}

public static long toUnsignedLong(int x) {
	return ((long) x) & 0xffffffffL;
}
```
`divideUnsigned` 和 `remainderUnsigned`分别是求两个`int`数作为无符号数相除得到的商和余。原理是将两个数转换为无符号`long`型（高32位填`0`）相除和求余。

# 获取指定的int型系统变量

TODO...