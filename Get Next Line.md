# open()
*syscall*
The `open()` function in C is a low-level system call used to open files for reading, writing, or both. It is declared in the header `<fcntl.h>` with the prototype:

```c
int open(const char *pathname, int flags, mode_t mode);
```

Unlike the higher-level `fopen()` from the standard library, `open()` communicates directly with the operating system to access files. It returns a file descriptor, a non-negative integer that uniquely identifies the opened file within the process. On failure, it returns -1.

**Parameters:**

- `pathname`: the path to the file to open.
- `flags`: controls how the file is opened. Common options:
    - `O_RDONLY`: open for reading only.
    - `O_WRONLY`: open for writing only.
    - `O_RDWR`: open for both reading and writing.
    - `O_CREAT`: create the file if it doesn't exist (requires the `mode` argument).
    - `O_TRUNC`: truncate the file to zero length if it exists (make it empty).
    - `O_APPEND`: append new data to the end of the file.
- `mode`: (optional) sets the file permissions when `O_CREAT` is used.
    - Example: `0644` (the owner can read and write; others can only read). The requested permissions are **always** filtered by the current `umask`.

---

# close()
*syscall*
The `close()` function in C is a low-level system call used to close an open file descriptor previously obtained through a function such as `open()`. It is declared in the header `<unistd.h>` with the prototype:

```c
int close(int fd);
```

The parameter `fd` is the file descriptor returned by `open()`. When a file is opened, the operating system allocates resources to manage it, and calling `close(fd)` releases those resources once the file is no longer needed. If the function succeeds, it returns 0. If an error occurs (for example, the file descriptor is invalid or already closed), it returns -1 and `errno` is set to indicate the specific error.

# perror()
*function*
The `perror()` function in C is a standard library function used to display a descriptive, human-readable message for the most recent system error. It is declared in the `<stdio.h>` header with the prototype:

```c
void perror(const char *s);
```

When a system call or library function (such as `open()`, `read()`, or `write()`) fails, it usually returns -1 and sets the global variable `errno` to indicate the type of error. The `perror()` function reads this `errno` value and prints an appropriate error message to the standard error stream (`stderr`).

The output format is `s: message`, where `s` is the custom string you pass to `perror()`. If `s` is `NULL` or an empty string, only the error message is printed. Using `perror()` is helpful for debugging and diagnosing runtime errors, as it clearly explains why a system call or file operation failed, such as "Permission denied" or "No such file or directory".

---

## Code Example Of Both

```c
#include <fcntl.h>   // for open() and its flags
#include <unistd.h>  // for write() and close()
#include <stdio.h>   // for perror() and printf()

int main(void)
{
    int the_return = open("example.txt", O_RDWR, O_CREATE, 0644);

    if (the_return == -1)
    {
        perror("7li 3ink");  // the error message that will be shown in the terminal
        return (1);
    }

    write(the_return, "Hello World", 11);
    printf("%d", the_return);
    close(the_return);

    return (0);
}
```

#### **1. Why does** `0644` **start with** `0`**?**

In C, a number starting with `0` is interpreted as an **octal number (base 8)**.

C supports different number bases:

|Prefix|Base|Example|
|---|---|---|
|No prefix|Decimal (base 10)|`644`|
|`0`|Octal (base 8)|`0644`|
|`0x`|Hexadecimal (base 16)|`0x644`|
|`0b`|Binary (base 2, in modern C)|`0b101`|

Octal uses digits from `0` to `7`, not `0` to `9`.

For permissions, octal is convenient because each digit represents three permission bits.
 - owner has 6: read and write -> 4 + 2 = `6`
 - user has 4: read only 
 - others has 4: read only too
### umask

`umask` (user file-creation mode mask) is a Linux mechanism that controls which permissions are removed when a new file or directory is created. Think of it as a security filter between the permissions you request and the permissions Linux actually gives your file.

- 1. How does it work?

Imagine you create a file using:

```c
open("file.txt", O_CREAT | O_WRONLY, 0777);
```

You request `0777`, meaning:

```
Owner    Group    Others
rwx      rwx      rwx
```

But Linux checks your `umask` before creating the file. Let's say the `umask` command returns:

```
0022
```

Linux removes the permissions specified by the mask.

**Example: `0777` with `umask 0022`**

| Value                 |        |
| --------------------- | ------ |
| Requested permissions | `0777` |
| Umask                 | `0022` |
| Actual permissions    | `0755` |

| Owner | Group | Others |
| ----- | ----- | ------ |
| `rwx` | `r-x` | `r-x`  |

**Why?**

`0022` means:

- First `0`: the special permissions mask.
- Second `0`: remove nothing from the owner.
- Third `2`: remove write permission from the group.
- Fourth `2`: remove write permission from others.

- 2. How does Linux calculate it?

The calculation is a bitwise operation:

```c
actual_permissions = requested_permissions & ~umask;
```

For example:

```
Requested:  0777
Umask:      0022
Result:     0755
```

**Important:** this is not ordinary subtraction. It is a bitwise AND combined with a bitwise NOT.

- 3. Why does `umask` exist?

Security! Imagine if every file you created automatically gave everyone permission to modify it. That would be dangerous. With `umask 0022`, newly created files won't be writable by other users.

---
# read()
*syscall*
The `read()` function in C is a low-level POSIX system call used to read raw bytes from a file descriptor rather than from `buffered FILE` streams. It is declared in the header `<unistd.h>` with the prototype:

c

```c
ssize_t read(int fd, void *buf, size_t count);
```

It takes three parameters:

- `fd`: a file descriptor, such as `0` for `stdin`, `1` for `stdout`, `2` for `stderr`, or any descriptor returned by `open()`.
- `buf`: a pointer to a buffer where the data will be stored.
- `count`: the maximum number of bytes to read.

When called, `read()` attempts to read up to `count` bytes, but it may return fewer depending on conditions such as reaching the end of the file, data not being immediately available (as with pipes or terminals), or partial reads imposed by the filesystem.

**Return value:**

- A positive number: that many bytes were successfully read.
- `0`: end-of-file.
- `-1`: an error occurred, and `errno` is set.

Because `read()` only copies raw bytes and does not append a null terminator, the caller must add `'\0'` manually if the data is to be treated as a string.

- **Steps to test `read()`**:
1. Open a file.
    
2. Read bytes from that file into a buffer.
    
3. Print what you read.
    
4. Close the file.
```c
#include <fcntl.h> //for open()

#include <unistd.h> //for close() and read()

#include <stdio.h> //for perror()

  

int main(void)

{

char buf[10];

ssize_t bytes;

int re;

re = open("t.txt", O_RDWR);

if (re == -1)

return (1);

write(re, "hey there",9);

lseek(re, 0, SEEK_SET);

re = open("t.txt", O_RDWR);

bytes = read(re, buf, 9);

if (bytes == -1)

return (1);

buf[bytes] = '\0';

printf("the text read: %s\n", buf);

printf("how many bytes: %zd",bytes);

close(re);

}
```

- **lseek():**
Every open file has a current offset that determines where the next `read()` or `write()` will happen, and `lseek()` lets you move it without reading or writing any data.It is declared in the header `<unistd.h>`
`lseek()` function in C is a low-level POSIX system call used to change the position of the file offset (the “cursor”) of an open file descriptor.
   -**Prototype:**

c

```c
off_t lseek(int fd, off_t offset, int whence);
```

**Parameters:**

- `fd`: the file descriptor of the open file, such as one returned by `open()`.
- `offset`: the number of bytes to move the offset. It can be negative to move backward.
- `whence`: defines the starting point from which `offset` is applied:
    - `SEEK_SET`: from the beginning of the file.
    - `SEEK_CUR`: from the current offset.
    - `SEEK_END`: from the end of the file.
### what is a buffer?
A **buffer** is a temporary area of memory used to hold data while it is being moved from one place to another. Instead of handling data one byte at a time, a program collects it in the buffer and processes it in larger chunks.

- **Two kinds of buffering** 

|                 | Where the buffer is                     | Who manages it               |
| --------------- | --------------------------------------- | ---------------------------- |
| Your own buffer | The array you declare (`char buf[100]`) | You                          |
| Stream buffer   | Inside a `FILE *`                       | The C library, automatically |
