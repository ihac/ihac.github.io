---
title: Buffer Overflow理论
date: 2016-05-03 10:02:52
tags: [Security, Assembly, Buffer Overflow]
---

> MIT Opencourseware
> Computer System Security 6-858

## Intro

Consider the following simplified example code from a Web server.
```c
int read_req() {
    char buf[128];
    int i;
    gets(buf);
    i = atoi(buf);
    return i;
}
```

It's dangerous because `gets()` has no bounds check. Thus, stack may be flushed by a long(more than 128 bytes) input sequence.

<!--more Keep on reading-->

Look as the architecture of stack in X86:
``` bash
                 +----------------+
entry %ebp ----> |.. prev frame ..|
                 |                |
                 |                |
                 +----------------+
entry %esp ----> | return address |
                 +----------------+
new %ebp   ----> |   saved %ebp   |
                 +----------------+
                 |       i        |
                 +----------------+
                 |    buf[127]    |
                 |      ...       |
                 |     buf[0]     |
                 +----------------+
new %esp   ----> |      ...       |
                 +----------------+
```

``` bash
# read_req's code:
    push %ebp
    mov %esp -> %ebp
    sub 168, %esp
    ... # program
    mov %ebp -> %esp
    pop %ebp
    ret
```

Since `gets()` has no bounds check, all data in stack might be overwrited by input sequence. Attackers can exploit this weakness and modify the `return address` of this function. The new `return address` may point to a uncharted area or even a malicious program.

## Solutions
### Stack Canaries

``` bash
    +------------+
    |  ret addr  |
    +------------+  ^
    | saved %ebp |  |
    +------------+  |
    |   CANARY   |  |  Overflow goes
    +------------+  |  this way.
    |  buf[127]  |  |
    |    ...     |  |
    |   buf[0]   |  |
    +------------+
```

`Canary` goes `in front of` return address on the stack, so that any overflow which rewrites the return address will also rewrite canary. Therefore, we can check `canary` before we `ret` from the function.

> Care that here we assume attackers can only rewrite the stack in a linear way
> That is to say, before rewriting ret addr, canary must be changed.

Another important thing is that `canary` cannot be easily guessed by attackers. It is interesting that `canary` can use `0`, `CR`, `LF` or `-1`, since these characters are terminated characters.

#### When do canaries fail

1. Overwrite function pointers before `canary`
2. `malloc()` and `free()`
```c
    char *p, *q;
    p = malloc(1024);
    q = malloc(1024);
    strcpy(p, buf);
    free(q);
    free(p);
```

    Assume that `p` and `q` are nearby.
Here is architecture of memory block:
``` bash
#   Allocated block        Free block
    +-----------+          +------------+
    |           |          |    size    |
    |   Data    |          +------------+
    |           |          |  ...empty  |
    +-----------+          +------------+
    |   Size    |          |  bkwd ptr  |
    +-----------+          +------------+
                           |   fwd ptr  |
                           +------------+
                           |    size    |
                           +------------+
```

    The buffer overrun in `p`(`strcpy()`) will overwrite the size value in `q`'s memory block!

    When `free()` merges two adjacent free blocks, it needs to manipulate `bkwd` and `fwd` pointers, and pointer calculation uses size to determine where the free memory block structure lives!
``` c
    p = get_free_block_struct(size);
    bck = p->bk;
    fwd = p->fd;
    fwd->bk = bck; // Writes memory
    bck->fd = fwd; // Writes memory
```
    By corrupting the `size` value, attackers can force `free()` to operate on a fake struct that resides in attacker-controlled memory and has attacker-controlled values for `forward` and `backward` pointers.
[reference](http://www.win.tue.nl/~aeb/linux/hh/hh-11.html)

### Bounds Check

- **Eletric fences**
Align each heap object with a guard page, and use page tables to ensure that accesses to the guard page cause a fault
``` bash
    +----------+
    |  Guard   |
    |          |  ^
    +----------+  |  Overflows cause
    |   Heap   |  |  a page exception
    |   obj    |  |
    +----------+
```
    Eletric fences can't protect the stack, and the memory overhead is too high to use in production systems.

- **Fat pointer**
Modify the pointer representation to include bounds information. Now, a pointer includes a memory address and bounds information about an object that lives in that memory region.
``` bash
// Example:
    /* Regular 32-bit pointer */
    +----------------+
    | 4-byte address |
    +----------------+

    /* Fat pointer */
+-----------------+----------------+---------------------+
| 4-byte obj_base | 4-byte obj_end | 4-byte curr_address |
+-----------------+----------------+---------------------+
```
    The program will be aborted if it dereferences a pointer whose address is outside of its own base-end range.
However, it can be expensive to check all pointer dereferences; Also, fat pointers are incompatible with a lot of existing software.

- **Baggy Bounds**
For each alocated object, store how big the object is.
There are 5 tricks in `Baggy Bounds`:
    1. Round up each allocation to a power of 2, and align the start of the allocation to that power of 2
    2. Express each range limit as log<sub>2</sub>(alloc_size). For 32-bit pointers, only need 5 bits to express the possible ranges.
    3. Store limit info in a linear array: fast lookup with one byte per entry. Also, we can use virtual memory to allocate the array on-demand.
    4. Allocate memory at slot granularity(e.g. 16bytes): fewer array entries
``` bash
    /* Memory Block */
    +-------+-------+-------+-------+
    |                               |
    +-------+-------+-------+-------+
    0       32      64      96     128

    a = malloc(28);
    +-------+-------+-------+-------+
    |  a    |                       |
    +-------+-------+-------+-------+
    0       32      64      96     128

    b = malloc(50);
    +-------+-------+-------+-------+
    |  a    |       |       b       |
    +-------+-------+-------+-------+
    0       32      64      96     128

    c = malloc(20);
    +-------+-------+-------+-------+
    |  a    |   c   |       b       |
    +-------+-------+-------+-------+
    0       32      64      96     128
```

    How we check the bounds:
``` c
    slot_size = 16;
    p = malloc(16);     table[p/slot_size] = 4;

    p = malloc(32);     table[p/slot_size] = 5;
/* 2 slot are used */   table[p/slot_size + 1] = 5;
    // Care that p is not the base address of the allocated block.

    // C code
    p1 = p + i;
    // Bounds check
    size = 1 << table[p >> log_of_slot_size];
    /** For example
     * p = malloc(28); slot_size = 16;
     * So table[p/slot_size] = table[p/slot_size + 1] = 5;
     * table[p >> log_of_slot_size] = 5;
     * size = 1 << 5 = 32
     */

    base = p & ~(size - 1);
    /** For example
     * size = 32 = 00100000; size - 1 = 00011111;
     * Assume p1's value(memory address) = 13 = 00001101
     * p & ~(size - 1) = 00001101 & 11100000 = 0
     * Assume p1's value(memory address) = 39 = 00100111
     * p & ~(size - 1) = 00100111 & 11100000 = 00100000 = 32
     */

    // check
    (p1 >= base) && ((p1 - base) < size)
                   or
    (p ^ p1) >> table[p >> log_of_slot_size] == 0
```
    5. Use virtual memory system to prevent out-of-bound derefs: set most signifcant bit in an OOB pointer, and then mark pages in the upper half of the address space as inaccessible. So we don't have to instrument pointer dereferences to prevent bad memory accesses.

　　Below is an example(assume slot_size = 16)
``` c
    char *p = malloc(44);
    // 64 bytes are allocated
    // 64 / slot_size = 4 bounds table entries
    // These entries are set to 6

    char *q = p + 60;
    // Access is ok!

    char *r = q + 16；
    // r is now at an offset of 60 + 16 = 76 from
    // p. This means 12 bytes beyond the end of p.
    // So baggy bounds will raise an error.

    char *s = q + 8;
    // s is now at an offset of 60 + 8 = 64 from
    // p. This means 4 bytes beyond the end of p,
    // which is less than half a slot away.
    // No error is raised, but the OOB high-order bit
    // is set in s, so that s cannot be dereferenced.

    char *t = s - 32;
    // t is now back inside the bounds,
    // so the OOB bit is cleared.
```

