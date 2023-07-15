---
title: "Z80 MasterClass: Case Statement in the Assembly Language"
tags: ["z80", "part time nerds", "iskra delta", "zx spectrum",
       "programming", "assembly", "masterclass"]
---


# Z80 MasterClass: Case Statement in the Assembly Language

[oddbit-retro, 11.2.2023](https://www.oddbit-retro.org/z80-masterclass-the-automata/)

A well-planned design can make your job a whole lot easier. Just take a look at how this *case* statement in assembly streamlines things, for instance.

~~~asm
        ld      a,(hl)                  
        cp      #' '
        jr      z,is_space
        cp      #'_'
        jr      z,is_underscore
        cp      #'$'
        jr      z,is_dollar
        jr      error_handler
~~~

The code is well-organized and straightforward. This is due to the fact that the compare instruction outputs the result in the `Z` flag while retaining the value of the `A` register. But what happens if you need to determine if a value falls within a specific range? Using the `CP` instruction would disrupt the natural flow, leading to jumps and increased complexity.

So here's a solution. Let's make our function act like the `CP` instruction, preserve the `A` register, and set or reset the `Z` flag as needed.

The function below does exactly that. It takes the symbol stored in the `A` register and checks if it falls within the bounds defined by the `D` and `E` registers (i.e. **`D >= A >= E`**). The result is then returned in the `Z` flag.

~~~asm
        ;; test if a is within DE: D >= A >= E
        ;; input(s):
        ;;  A   value to test (preserved!)
        ;;  DE   interval
        ;; output(s):
        ;;  Z    zero flag is 1 if inside, 0 if outside
        ;; affects:
        ;;  D, E, flags
test_inside_interval:
        push    bc                      ; store original bc
        ld      c,a                     ; store a
        cp      e			            ; a=a-e
        jr      nc, tidg_possible	    ; a>=e       
        jr      tidg_false              ; false
tidg_possible:
        cp      d                       ; a=a-d
        jr      c,tidg_true		        ; a<d
        jr      z,tidg_true             ; a=d
        jr      tidg_false
tidg_true:
        ;; set zero flag
        xor     a                       ; a=0, set zero flag
        ld      a,c                     ; restore a
        pop     bc                      ; restore bc
        ret
tidg_false:
        ;; reset zero flag
        xor     a
        cp      #0xff                   ; reset zero flag
        ld      a,c                     ; restore a
        pop     bc                      ; restore original bc
        ret
~~~

We can easily create more functions like this by simply filling in the `DE` and `A` registers and linking the calls together smoothly, just like in the `test_if_digit` function below. 

~~~asm
test_is_digit:
        ld      de,#0x3930	            ; d='9', e='0'
        jr      test_inside_interval    ; ret optimization...
~~~

It's worth noting that this function does not end with the `ret` instruction. Instead, it jumps to the next function. And once that next function returns, it will proceed straight to the callee, saving us one jump.

By using this technique, we can optimize our code. The following code snippet implements several additional functions, all utilizing the same `ret` instruction.

~~~asm
test_is_alpha:
        call    test_is_upper
        ret     z
        jr      test_is_lower

test_is_upper:
        ld      de,#0x5a41              ; d='Z'. e='A'
        jr      test_inside_interval

test_is_lower:
        ld      de,#0x7a61              ; d='z', e='a'
        jr      test_inside_interval    ; last tests' result is the end result

test_is_alphanumeric:
        call    test_is_digit
        ret     z
        jr      test_is_alpha
~~~

In conclusion, if your comparison functions mimic the `CP` instruction, you can link them together and write sophisticated *case* statements that seamlessly integrate with the assembly language.

~~~asm
        ld      a,(hl)              
        call    test_is_digit           
        jr      z,is_digit
        cp      #' '
        jr      z,is_space
        cp      #'_'
        jr      z,is_underscore
        call    test_is_alpha
        jr      z,is_alpha
        jr      error_handler
~~~

For a practical demonstration of this approach, check out [this real-world example](https://github.com/tstih/idp-udev/blob/main/src/ulibc/ctype.s) - it's a partially implemented version of the ctype.h standard C library functions.