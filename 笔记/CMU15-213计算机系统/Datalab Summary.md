## Datalab Summary

### 1、bitXor

```c++
要求：实现异或
分析：采用德摩根律即可
代码：
int bitXor(int x, int y) {
  return ~((~x)&(~y))&(~(x&y));
}
```

### 2、tmin

```c++
要求；返回Tmin
代码：
int tmin(void) {
  return 1<<31;
}
```

### 3、isTmax

```c++
要求：如果是Tmax，则返回1
分析：tmax+1再取反为tmax本身，另外-1也符合此情况，需将-1排除
代码：
int isTmax(int x) {
  return !(~(x+1)^x)&!!(x+1);
}
```

### 4、allOddBits

```c++
要求：当所有奇数位为1时返回1
分析：先取所有奇数位，再与0xAAAAAAAA相异或
代码：
int allOddBits(int x) {
  return !((x&(0xAAAAAAAA))^(0xAAAAAAAA));
}
```

### 5、negate

```c++
要求：返回-x
代码：
int negate(int x) {
  return (~x)+1;
}
```

### 6、isAsciiDigit

```c++
要求：return 1 if 0x30 <= x <= 0x39
分析：
1、x前26位为0
2、x的第27-28位为11
3、x的末4位减0xA为负值
代码：
int isAsciiDigit(int x) {
     int a = !(x >> 6);
     int b = !((x >> 4) ^ (0b11));
     int c = 0xF & x;
     int d = ((~0xA) + 1) + c;
     int e = !!(d >> 31);
  return a & b & e;
}
```

### 7、conditional

```C++
要求：实现x ? y : z
分析：x为非0取值y，x为0取值z
代码：
int conditional(int x, int y, int z) {
        int mask = (!!x) << 31 >> 31;
        return (mask & y) | (~mask & z);
}
```

### 8、isLessOrEqual

```c++
要求：当x<=y时返回1，否则返回0
分析：当x和y异号时，x必须为负，当x和y同号时，y-x必须>=0
代码：
int isLessOrEqual(int x, int y) {
        int sx = x >> 31;
        int sy = y >> 31;
        int dif = sx ^ sy;
        int sub = (~x+1)+y;
        return (((!dif&!(sub >> 31))|(dif&sx))) & 1;
}
```

### 9、logicalNeg

```c++
要求：实现！运算符
分析：这里需要一个小技巧：只有当x=0时，x和-x的符号位相同且为0
代码：
    int logicalNeg(int x) {
        return (~((~x+1)|x)>>31)&1;
	}

```

### 10、howManyBits

```c++
要求：返回表示一个数所用的补码的最少位数
分析：如果是正数，则为最左侧1的位数加1，如果是负数，则将它反转后采用同样的方法
代码：
int howManyBits(int x) {
        int mask = x >> 31;
        x = (mask&(~x))|(~mask&x);//实现负数反转功能
        //以下为一个取各个位置的小技巧
    	int b16,b8,b4,b2,b1,b0;
        b16 = !!(x >> 16) << 4;
        x >>= b16;
        b8 = !!(x >> 8) << 3;
        x >>= b8;
        b4 = !!(x >> 4) << 2;
        x >>= b4;
        b2 = !!(x >> 2) << 1;
        x >>= b2;
        b1 = !!(x >> 1);
        x >>= b1;
        b0 = x;
        return 1 + b0 + b1 + b2 + b4 + b8 + b16;
	}
```

### 11、floatScale2

```c++
要求：返回2*f
分析：
	当exp = 0xff时（数为NAN或无穷大），返回原值;
	当exp为0时，返回非规格化值
	当exp为其他时，返回规格化值
代码：
        unsigned floatScale2(unsigned uf) {
        unsigned s = (uf >> 31) & 1;
        unsigned exp = (uf >> 23) & 0xff;
        unsigned frac = uf & 0x7fffff;
        unsigned res;
        if (exp == 0xff) {
                return uf;
        } else if (exp == 0) {
                frac <<= 1;
                res = (s << 31) | (exp << 23) | frac;
        } else {
                exp++;
                res = (s << 31) | (exp << 23) | frac;
        }
        return res;
	}

```

### 12、floatFloat2Int

```c++
要求：返回float转换的int形式
分析：
	当E >= 31时，返回0x80000000u;
	当E < 0时，返回0;
	当0 <= E < 31时，转换成int形式;
代码：
    int floatFloat2Int(unsigned uf) {
        int s = (uf >> 31) & 1;
        unsigned exp = (uf >> 23) & 0xff;
        unsigned frac = uf & 0x7fffff;
        int E = exp - 127;
        if (E >= 31) {
                return 0x80000000u;
        } else if (E < 0){
                return 0;
        } else {
                frac = frac | (1 << 23);//将frac还原成1.xxx形式
                //当E < 23时，需要右移舍入
            	if (E < 23) {
                        frac >>= (23 - E);
                } else {
                        frac <<= (E - 23);
                }
                if (s == 0) return frac;
                else return -frac;
        }
	}

```

### 13、floatPower2

```c++
要求：返回2^x次方的float表示（用unsigned返回）
分析：
    x > 127时，返回+INF;
    -126 <= x < 127时，返回规格化数;
    -147 <= x < -126时，返回非规格化数;
    x <= -148时，返回0;
代码：
    unsigned floatPower2(int x) {
        if (x > 127) {
            return 0xff << 23;
        } else if (x >= -126) {
            int exp = x + 127;
            return exp << 23;
        } else if (x >= -147) {
            int t = 148 + x;
            return 1 << t;
        } else {
            return 0;
        }
    }
```

