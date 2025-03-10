# SD Lab 2: Secure Coding

## Completed by Ferodov Alexey (tg: @ullibniss)

---

# Description

```
Objectives

In this lab you will:
1. Practice reading program code and finding potential security breaches in it.
2. Explain how exactly do your findings threaten security of the analyzed code.
3. Categorize each finding as threatening either integrity, confidentiality, or availability.
4. Fix each finding in the code; explain how your modification fixes the problem.
5. Introduce hardenings to avoid the risks created by similar problems.

Description
You are given an archive with two files, hash.c and hash.h, that contain a naive hash table
implementation. The code contains programming mistakes that make it vulnerable in some way. You
will need to:
```

# 1 Copy hash.c and hash.h to hash_fixed.c and hash_fixed.h, respectively. 

Let's begin. I copied files as needed.

![image](https://github.com/user-attachments/assets/6cbcdf84-f19d-48cc-89c6-9832a152ed0c)

# {2,3}. Find as many security related programming mistakes as you can in hash.c and hash.h. For every mistake you find (comment directly in the code for each step).

```
a. Clearly locate the mistake in the code.
b. Explain why it is a mistake and how it affects security of the code.
c. Categorize the mistake as affecting integrity, confidentiality, or availability.
d. Fix the mistake in hash_fixed.c and hash_fixed.h.
e. Explain why do you expect your fix to remove the problem.
f. Introduce a hardening to avoid the risks created by the mistake.
```

## List of Mistakes

### 1. `HashIndex` function infinite loop (CWE-835)

#### Location

`hash.c` - line 14.

```
unsigned HashIndex(const char* key) {
    unsigned sum = 0;
    for (char* c = key; c; c++){
        sum += *c;
    }
    return sum;
}
```

#### Problem

The loop in `HashIndex` has an infinite loop condition because `c` is a `char*` and the loop condition checks `c`, which is a **pointer**, but it should be `*c`. Incorrect condition in the loop may lead to unexpected behavior, such as an infinite loop or accessing memory outside the intended bounds, potentially resulting in memory corruption.

#### Category

**Integrity**. Because it could corrupt the function's ability to properly generate hash indices.

#### Fix

```
unsigned HashIndex(const char* key) {
    unsigned sum = 0;
->  for (const char* c = key; *c != '\0'; c++){
        sum += *c;
    }
    return sum;
}
```

The condition now properly checks for the null-terminator (\0) of the string, preventing an infinite loop.

#### Hardering

Always verify string termination by checking for the null-terminator (\0) when processing strings in C to avoid infinite loops and memory corruption.

### 2. `HashIndex` function. Using an incorrect data type for character summation (CWE-190)

#### Location

`hash.c` - line 15

```
unsigned HashIndex(const char* key) {
    unsigned sum = 0;
    for (const char* c = key; *c != '\0'; c++){
->      sum += *c;
    }
    return sum;
}
```

#### Problem

`sum += *c;` adds the character to the sum, but `*c` is of type `char`, which may be `signed` or `unsigned` depending on the platform. This could lead to unexpected behavior if the character value is negative, affecting the hash calculation and possibly causing an overflow.

Incorrect handling of character values can provide overflow or incorrect hash values. This affects the integrity of the hash function and may result in collisions, which can be exploited in hash-based data structures.

#### Category

**Integrity** - because the hash calculation is incorrect.

#### Fix

```
unsigned HashIndex(const char* key) {
    unsigned sum = 0;
    for (const char* c = key; *c != '\0'; c++){
        sum += (unsigned char)*c;
    }
    return sum;
}
```

Fix prevents potential integer overflows or unexpected values by casting to unsigned char.

#### Hardering

Always cast `char` to `unsigned char` when performing arithmetic operations on character values to ensure predictable behavior across platforms.

### 3 Use of `strcpy` for string comparison in `HashFind` and `HashDelete` (CWE-804)

#### Location

`hash.c` - line 37
`hash.c` - line 48

```
PairValue* HashFind(HashMap *map, const char* key) {
    unsigned idx = HashIndex(key);
    
    for( PairValue* val = map->data[idx]; val != NULL; val = val->Next ) {
->      if (strcpy(val->KeyName, key))
            return val;
    }
    
    return NULL; 
}

void HashDelete(HashMap *map, const char* key) {
    unsigned idx = HashIndex(key);
    
    for( PairValue* val = map->data[idx], *prev = NULL; val != NULL; prev = val, val = val->Next ) {
->      if (strcpy(val->KeyName, key)) {
            if (prev)
                prev->Next = val->Next;
            else
                map->data[idx] = val->Next;
        }
    }
}
```

#### Problem

`strcpy` is used for string comparison, which is incorrect. strcpy copies the source string into the destination, and the condition will always evaluate to the address of the copied string, which is not a valid comparison.  The use of `strcpy` instead of `strcmp` can lead to unexpected behavior and, in some cases, buffer overflows if the source string is larger than the destination buffer.

#### Category

**Integrity** - because it compromises the logic of finding and deleting keys correctly.

#### Fix

```
if (strcmp(val->KeyName, key) == 0)
```

The correct function to compare strings is `strcmp`. It compares two strings and returns 0 if they are equal. This fix ensures that we properly check if the `KeyName` matches the input `key`.

#### Hadrering

Ensure that the `KeyName` buffer is properly sized and null-terminated before performing the comparison to prevent buffer overflows or undefined behavior.'

### 4 No memory deallocation in HashDelete ( CWE-401 my favourite)

#### Location

`hash.c` - line 53

```
    for( PairValue* val = map->data[idx], *prev = NULL; val != NULL; prev = val, val = val->Next ) {
        if (strcmp(val->KeyName, key) == 0) {
            if (prev)
                prev->Next = val->Next;
            else
                map->data[idx] = val->Next;
->      }
    }
```

#### Problem

There is no memory deallocation for the `PairValue` object being removed from the hash map. This leads to a **memory leak**, where memory that is no longer in use is not freed. Memory leaks can lead to exhaustion of available memory, causing the application to crash or behave unpredictably. It also increases the attack surface by leaving sensitive data in memory.

#### Category

**Availability** - because memory leaks can lead to resource exhaustion.

#### Fix

```
    for( PairValue* val = map->data[idx], *prev = NULL; val != NULL; prev = val, val = val->Next ) {
        if (strcmp(val->KeyName, key) == 0) {
            if (prev)
                prev->Next = val->Next;
            else
                map->data[idx] = val->Next;
            free(val);
        }
    }
```

The `free(val)` call ensures that the memory allocated for the `PairValue` structure is deallocated properly when it is removed from the hash map. This prevents memory leaks and ensures that system resources are used efficiently.

#### Hardering

Ensure that all dynamically allocated memory is freed when it's no longer needed. Implement a memory management strategy that includes tracking memory allocations and deallocations. It also might be useful to test program in `Valgrind`.

### 5 No NULL pointer checks before dereferencing

#### Location

`hash.c` - line 37

```
    for( PairValue* val = map->data[idx]; val != NULL; val = val->Next ) {
->      if (strcmp(val->KeyName, key) == 0)
            return val;
    }
```

#### Problem

While this code checks for `NULL` on the initial `map->data[idx]`, the next `Next` pointer access may not handle all cases properly, especially if `val->Next` is `NULL` but `val` is still valid. Further, if `map->data[idx]` is NULL, dereferencing `val` could be unsafe in other parts of the code.

Dereferencing `NULL` or uninitialized pointers can cause crashes or memory corruption, which can be exploited by attackers for various attacks, such as `Denial-of-Service`.

#### Fix

```
for (PairValue* val = map->data[idx]; val != NULL; val = val->Next) {
    if (val->KeyName != NULL && strcmp(val->KeyName, key) == 0) {
        return val;
    }
}
```

The fix adds a check to ensure that val->KeyName is not NULL before attempting a comparison. This prevents dereferencing a NULL pointer, which can cause a crash.

#### Hardering

Always ensure that pointers are validated and that `NULL` checks are performed before dereferencing pointers in the code.

