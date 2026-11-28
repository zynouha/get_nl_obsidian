# `ps` and `lsof -p`: finding a process and inspecting its open files

## Step 1: `ps` finds the process

`ps` stands for **process status**. It prints a snapshot of the processes running at that moment. `ps` comes first because `lsof -p` needs a **PID**, and `ps` is how you find it.

```bash
ps
```

With no options, `ps` shows only the processes started from your current terminal, usually your shell and `ps` itself:

```
    PID TTY          TIME CMD
   1234 pts/2    00:00:00 fish
   5678 pts/2    00:00:00 ps
```

##### **1. PID — Process ID**

```
1234
5678
```

The unique identifier assigned to each process by the operating system.

- `1234` → PID of your Fish shell.
    
- `5678` → PID of the `ps` command.
    

You can use the PID to inspect or send signals to a process.

##### **2. TTY — Controlling Terminal**

```
pts/2
pts/2
```

TTY stands for **Teletypewriter**.
##### **3. TIME**
This represents the total CPU time consumed by the process.

**Important:** It is NOT the time since the process started.

For example, imagine your C program runs for 10 minutes but spends most of that time sleeping:

```
while (1)
    sleep(1);
```

Its CPU time might remain very small because sleeping doesn't continuously consume CPU.
CPU time measures time spent executing on the CPU, not elapsed wall-clock time.
##### **4. CMD — Command**
This tells you the command associated with the process.

- `fish` → Your Fish shell.
    
- `ps` → The process listing command itself.
    

Notice that `ps` appears in its own output! That's because `ps` is itself a running process when it collects the process information.

**ps** tells you which terminal is associated with the process.
Once you have the PID, give it to `lsof -p`.

---

## Step 2: `lsof -p` inspects the process's open files

`lsof` stands for **List Open Files**. On Linux, almost everything is a file (regular files, directories, terminals, pipes, sockets), so `lsof` shows what a process is touching. The `-p` option limits the output to one process ID.

```bash
lsof -p PID

----------------------------------------------------------------
COMMAND    PID  USER   FD   TYPE DEVICE SIZE/OFF    NODE NAME
a.out   380467 nzayi  cwd    DIR    8,0     4096     512 /home/nzayi
a.out   380467 nzayi  rtd    DIR  252,0     4096       2 /
a.out   380467 nzayi  txt    REG    8,0    15920  100132 /home/nzayi/a.out
a.out   380467 nzayi  mem    REG  252,0  2220400 5648494 /usr/lib/x86_64-linux-gnu/libc.so.6
a.out   380467 nzayi  mem    REG  252,0   240936 5648488 /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
a.out   380467 nzayi    0u   CHR  136,1      0t0       4 /dev/pts/1
a.out   380467 nzayi    1u   CHR  136,1      0t0       4 /dev/pts/1
a.out   380467 nzayi    2u   CHR  136,1      0t0       4 /dev/pts/1

```


The **TTY** column is the same terminal device (`/dev/pts/2`) that appears in the `lsof` output later in this note.


> **Using fish?** `$$` is not the shell's PID in fish. Use `$fish_pid` instead: `lsof -p $fish_pid`. In bash or zsh, use `lsof -p $$`.

### Reading the output

|Column|Meaning|
|---|---|
|COMMAND|Process name|
|PID|Process ID|
|USER|Process owner|
|FD|File descriptor or special name (see below)|
|TYPE|`REG` regular file, `DIR` directory, `CHR` character device, `FIFO` pipe, `IPv4`/`IPv6`/`unix` sockets|
|DEVICE|Device numbers (major,minor)|
|SIZE/OFF|File size, or the current offset (shown as `0t...` when using `-o`)|
|NODE|Inode number|
|NAME|Path or name|
### Understanding Each Column:
##### **1. COMMAND**

```
a.out
```

The name of the command associated with the process.

In your case, `a.out` is the name of your compiled C executable.

**Important:** This is not necessarily the full path of the executable. It is usually the command name.

##### **2. PID**

```
380467
```

PID stands for **Process ID**.

Every running process has an identifier assigned by the kernel.

You can use it to:

- Inspect the process: `ps -p 380467`
    
- List its open files: `lsof -p 380467`
    
- Send it a signal: `kill 380467`
##### **3. USER**

```
nzayi
```

The username associated with the process.

It tells you which user owns/runs the process.
##### **4. The FD column**

Plain numbers are real descriptors in the process's FD table. A letter after the number shows the access mode.

| Value            | Meaning                                                               |
| ---------------- | --------------------------------------------------------------------- |
| `0r`, `1w`, `2u` | stdin, stdout, stderr (`r` = read, `w` = write, `u` = read and write) |
| `3r`             | fd 3, opened `O_RDONLY`                                               |
| `4w`             | fd 4, opened `O_WRONLY`                                               |
| `5u`             | fd 5, opened `O_RDWR`                                                 |
| `cwd`            | Current working directory                                             |
| `rtd`            | Root directory                                                        |
| `txt`            | The program's executable                                              |
| `mem`            | Memory-mapped file (shared libraries, for example)                    |
##### **5. TYPE**

This tells you what kind of object the process is accessing.

| Value  | Meaning            |
| ------ | ------------------ |
| `REG`  | Regular file       |
| `DIR`  | Directory          |
| `CHR`  | Character device   |
| `BLK`  | Block device       |
| `FIFO` | Named pipe         |
| `IPv4` | IPv4 socket        |
| `IPv6` | IPv6 socket        |
| `unix` | Unix domain socket |
The letter is the access mode stored in the **file table entry**, the same flags the kernel checks when you call `read()` or `write()`.

##### **6. DEVICE**

DEVICE answers one question: **"Which piece of hardware (or storage area) is this file stored on?"**
It is shown as two numbers, like `8,0`:
- **Major number:** Identifies the device driver or device class.
    
- **Minor number:** Identifies a particular device handled by that driver.
- 
##### **7. SIZE/OFF**
Example:

```
4096
15920
0t0
```

This column can represent either the file size or an offset, depending on the object and descriptor.
- `15920` represents the executable's size in bytes.
- `0t0` represents an offset of zero, expressed in decimal (`t`). 

##### **8. NODE**
An inode is a filesystem data structure containing metadata about a file, such as:

- File permissions.
    
- Owner.
    
- File size.
    
- Timestamps.
    
- Pointers/references to the file's data blocks.

**Important:** A filename is not the inode. The filename is a directory entry that refers to an inode.

You can inspect an inode using:

```
ls -i a.out
```
##### **9. NAME**
 Example:

```
/home/nzayi
/home/nzayi/a.out
/usr/lib/x86_64-linux-gnu/libc.so.6
/dev/pts/1
```

This identifies the file or resource associated with the process.

Depending on the type, it can be:

- A filesystem path.
    
- A terminal device.
    
- A socket address.
    
- A pipe identifier.
## **Let's put everything together**

**Your standard output entry**

```
a.out  380467  nzayi  1u  CHR  136,1  0t0  4  /dev/pts/1
```

1. **COMMAND:** `a.out`
    
    Which program?
    
2. **PID:** `380467`
    
    Which process?
    
3. **USER:** `nzayi`
    
    Which user?
    
4. **FD:** `1u`
    
    Which descriptor? Standard output, read/write mode.
    
5. **TYPE:** `CHR`
    
    What kind of resource? Character device.
    
6. **DEVICE:** `136,1`
    
    Which device?
    
7. **SIZE/OFF:** `0t0`
    
    What's its offset? Zero.
    
8. **NODE:** `4`
    
    What's its inode number?
    
9. **NAME:** `/dev/pts/1`
    
    Which resource exactly?