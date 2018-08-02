---
title: Redis源码阅读(一) 简单动态字符串
date: 2018-08-01 13:26:01
tags:
- redis
categories:
- 源码阅读
---

​Redis中自定义了简单动态字符串，兼容传统C字符串，并提供了一些高效、安全的方法，用于Redis键值对中需要字符串的存储。除了一些日志输出的字面量外，其余需要修改的字符串都以sds作为存储。

### 1. 定义

```c
typedef char* sds;

struct sdshdr64 {
    uint64_t len;		// 使用的大小
    uint64_t alloc;     // 分配的大小，不包括头部和空字符
    unsigned char flags;// 低3位用于表示header类型，其余5位未使用
    char buf[];
}
```

sds是char *的别名，指向了存储字符串数组的首地址。

sdshdr是存储的真正结构，包含len, alloc, flags, buf字段。

- len: 字符串长度，不包含终止字符串'\0'
- alloc: 字符串最大容量，不包含header和最后的'\0'
- flags: 低3bit用于表示header类型，高5bit未使用(sdshdr5中高5bit用于存储字符串长度)
- buf: 存放字符串的内存空间，sds指向该空间的首地址

为了节省内存，Redis实现了不同大小的sdshdr，用于存储不同长度的字符串

- sdshdr5: 0 ~ 31时使用，没有len,alloc字段，用flags的高5位表示字符串长度
- sdshdr8: 32 ~ 255 时使用
- sdshdr16: 256 ~ 65535 使用
- sdshdr32: 64位机器中，65536 ~ 2^32-1 使用
- sdshdr64: 32位机器中大于65535、64位机器中大于 2^32-1 使用

> 使用packed属性取消编译器对结构体的字节对齐

### 2. 优势

- 获取字符串长度 O(1)

  传统C字符串使用‘\0’表示字符结尾，使用strlen(s)获取字符串长度，遍历字符串数组并计数，遇到'\0'表示结束并输出字符串长度，时间复杂度为O(N)，SDS字符串在结构体里存储了字符串长度，获取为常数时间O(1)。

- 避免缓存区溢出

  传统C字符串使用strcat拼接时，由于不清楚字符串长度，可能会使用到非分配的内存空间，导致内存溢出，SDS字符串API保证了拼接字符串时若剩余空间不足，则会先分配内存空间，再进行拼接。

- 减少内存分配次数

  传统C字符串进行修改时，会重新分配内存空间，再进行数据拷贝，而SDS字符串采用预分配的方式，每次分配内存大小是需要的2倍(但每次分配不能大于1M，还有额外的'\0')，SDS字符串缩小时进行数据拷贝并修改len，不释放内存空间，保留原来分配的大小。

- 二进制安全

  传统C字符串只能包含特定编码的数据，而且不能含有空值('\0')，否则会当作字符串截断进行处理，导致C字符串只能保存文本数据，而对于二进制数据(图像、音频、视频等)则无能为力，SDS字符串使用len属性判断字符串结束，故能存储任何数据。

### 3. 源码剖析

- sds定义

```c
/*
 * src/sds.h
 */

// 使用char *存放字符串，可复用C语言字符串操作库函数，利用该指针指向sdshdr中buf字段。
typedef char *sds;   

// 使用__attribute__ ((__packed__)) 的作用是取消编译器对结构体进行字节对齐

// sds结构定义了5中header，分别用于不同长度字符串的存储，从而节省内存。
// len:	  字符串真正长度，不包含终止字符串
// alloc: 字符串最大容量，不包含header和最后终止字符串
// flags: 低3bit用于表示header类型，高5bit未使用
// buf:   连续空间存储，存放字符串的真正内存空间,sds就是指向了该空间的首地址

struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags;// 低3位用于表示header类型，高5位用于字符串长度存储
    char buf[];
}
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len;		// 使用的大小
    uint8_t alloc;      // 分配的大小，不包括头部和空字符
    unsigned char flags;// 低3位用于表示header类型，其余5位未使用
    char buf[];
}
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len;		// 使用的大小
    uint16_t alloc;     // 分配的大小，不包括头部和空字符
    unsigned char flags;// 低3位用于表示header类型，其余5位未使用
    char buf[];
}
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len;		// 使用的大小
    uint32_t alloc;     // 分配的大小，不包括头部和空字符
    unsigned char flags;// 低3位用于表示header类型，其余5位未使用
    char buf[];
}
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len;		// 使用的大小
    uint64_t alloc;     // 分配的大小，不包括头部和空字符
    unsigned char flags;// 低3位用于表示header类型，其余5位未使用
    char buf[];
}

// 用于获取sdshdr的指针,即指针按照头部大小向前偏移就能找到header的指针
# define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T)))) 
```

- 空间预分配，字符串拼接时内存空间不够

  - newlen*2 > SDS_MAX_PREALLOC(1MB)，则分配1MB内存空间
  - newlen*2 < SDS_MAX_PREALLOC, 则分配newlen两倍的内存空间

  若长度改变未导致sdshdr头部类型改变，则需要扩展内存空间大小，否则需要重新分配所有内存，将原有数据拷贝，再释放内存空间。

```c
/*
 * src/sds.c
 */

/**                                                                                                 
 * 扩大sds字符串内存空间，不会修改len大小                                                                            
 */                                                                                                 
sds sdsMakeRoomFor(sds s, size_t addlen) {                                                          
    void *sh, *newsh;                                                                               
                                                                                                    
    // 获取当前可用空间大小                                                                         
    size_t avail = sdsavail(s);                                                                     
    size_t len, newlen;                                                                             
    char type, oldtype = s[-1] & SDS_TYPE_MASK;                                                     
    int hdrlen;                                                                                     
                                                                                                    
    /* Return ASAP if there is enough space left. */                                                
    // 空间足够则不需要改变                                                                         
    if (avail >= addlen) return s;                                                                  
                                                                                                    
    len = sdslen(s);                                                                                
    // 获取sds字符串头部hdr                                                                         
    sh = (char*)s-sdsHdrSize(oldtype);                                                              
    // 需要的新空间大小                                                                             
    newlen = (len+addlen);                                                                          
    // 每次分配的为新空间大小的2倍，但不能超过每次最大分配内存(1MB)                                 
    if (newlen < SDS_MAX_PREALLOC)                                                                  
        newlen *= 2;                                                                                
    else                                                                                            
        newlen += SDS_MAX_PREALLOC;                                                                 
                                                                                                    
    // 根据新长度选择sdshdr类型                                                                     
    type = sdsReqType(newlen);                                                                      
                                                                                                    
    /* Don't use type 5: the user is appending to the string and type 5 is                          
     * not able to remember empty space, so sdsMakeRoomFor() must be called                         
     * at every appending operation. */                                                             
                                                                                                    
    // 不能使用hdr5,因为没有alloc字段表示剩余空间                                                   
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;                                                      
                                                                                                    
    // 头部长度                                                                                     
    hdrlen = sdsHdrSize(type);                                                                      
    if (oldtype==type) {                                                                            
        // 类型没变，则扩展内存空间                                                                 
        newsh = s_realloc(sh, hdrlen+newlen+1);                                                     
        if (newsh == NULL) return NULL;                                                             
        // 获取新sds字符串数组(buf)首地址                                                           
        s = (char*)newsh+hdrlen;                                                                    
    } else {          
         /* Since the header size changes, need to move the string forward,                          
         * and can't use realloc */                                                                 
        // header类型改变，重新分配内存                                                             
        newsh = s_malloc(hdrlen+newlen+1);                                                          
        if (newsh == NULL) return NULL;                                                             
        // 将字符串数组拷贝到新空间                                                                 
        memcpy((char*)newsh+hdrlen, s, len+1);                                                      
        // 释放之前的内存空间                                                                       
        s_free(sh);                                                                                 
        // 获取新sds字符串数组(buf)首地址                                                           
        s = (char*)newsh+hdrlen;                                                                    
        // 类型赋值                                                                                 
        s[-1] = type;                                                                               
        // 改变sds长度                                                                              
        sdssetlen(s, len);                                                                          
    }                                                                                               
    // 改变sds字符串alloc                                                                           
    sdssetalloc(s, newlen);                                                                         
    return s;                                                                                       
}
```

- 重新分配内存空间，保证没有浪费

```c
/**                                                                                                 
 * 重新分配内存空间，保证没有浪费                                                                   
 */                                                                                                 
sds sdsRemoveFreeSpace(sds s) {                                                                     
    void *sh, *newsh;                                                                               
    char type, oldtype = s[-1] & SDS_TYPE_MASK;                                                     
    int hdrlen, oldhdrlen = sdsHdrSize(oldtype);                                                    
    size_t len = sdslen(s);                                                                         
    sh = (char*)s-oldhdrlen;                                                                        
                                                                                                    
    /* Check what would be the minimum SDS header that is just good enough to                       
     * fit this string. */                                                                          
    // sds字符串类型和头部长度                                                                      
    type = sdsReqType(len);                                                                         
    hdrlen = sdsHdrSize(type);                                                                      
                                                                                                    
    /* If the type is the same, or at least a large enough type is still                            
     * required, we just realloc(), letting the allocator to do the copy                            
     * only if really needed. Otherwise if the change is huge, we manually                          
     * reallocate the string to use the different header type. */                                   
                                                                                                    
    // 若未改变类型或者需要一个足够大的类型, 则进行realloc, 否则malloc                                           
    if (oldtype==type || type > SDS_TYPE_8) {     
        // 在原有基础上改变内存大小
        newsh = s_realloc(sh, oldhdrlen+len+1);                                                     
        if (newsh == NULL) return NULL;     
        // 获取新sds字符串数组(buf)首地址    
        s = (char*)newsh+oldhdrlen;                                                                 
    } else {    
        // 完全重新分配内存空间
        newsh = s_malloc(hdrlen+len+1);                                                             
        if (newsh == NULL) return NULL; 
        // 拷贝数据
        memcpy((char*)newsh+hdrlen, s, len+1);       
        // 释放原有内存
        s_free(sh);   
        // 获取新sds字符串数组(buf)首地址    
        s = (char*)newsh+hdrlen;  
        // 新类型
        s[-1] = type;  
        // 长度
        sdssetlen(s, len);                                                                          
    }
    // avail为0
    sdssetalloc(s, len);                                                                            
    return s;                                                                                       
}
```

