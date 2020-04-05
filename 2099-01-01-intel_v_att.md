---
title:  "Quick Notes: Intel vs. At&t Syntax"
date:   2020-03-26 01:01:00
categories: notes quick
layout: post
---

Unfortunately, you are often going to see both At&t and Intel syntax to
represent assembly instructions, so you will have to be comfortable with both.
Some tools---such as GDB and objdump---use At&t syntax as the default while
others---e.g., IDA and PEDA---use Intel syntax.

Here are some of the major differences with examples:

```
                                          ATT           Intel
different order of operands               src, dst      dst, src
no reg. prefix in Intel                   %rbx          rbx
no size suffix in Intel                   movq          mov
different location descriptions           (%rbx)        QWORD PTR [rbx]
no prefix for immediate in Intel          $0x1F         0x1F
```

The language for sizes is also confusing. This chart is for x86-64. Pointers
(`char *`) are only 4 bytes in x86-32.

```
C          Intel                  ATT Suffix    Size (bytes)
char       byte                   b             1
short      word                   w             2
int        dbl word               l             4
long       quad word              q             8
char\*     quad word              q             8
float      single precision       s             4
double     double precision       l             8
``` 


