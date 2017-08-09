Actually everything related to generated machine code... not just codegen.

### The effect of MONO_DEBUG=disable_omit_fp.

Consider the following simple function:
```
frame #2: 0x0000000104cb1fe9 Mono`System.EmptyArray`1<T_REF>:.cctor () at EmptyArray.cs:33
   30   {
   31           static class EmptyArray<T>
   32           {
-> 33                   public static readonly T[] Value = new T [0];
   34           }
   35   }
 ```
<table>
<tr><th>With fp omission</th><th>Without fp omission</th>
<tr><td>
<pre>
Mono`System.EmptyArray`1<T_REF>:.cctor ():
    0x104cb1fd0 <+0>:  subq   $0x18, %rsp
    0x104cb1fd4 <+4>:  movq   %r10, (%rsp)
    0x104cb1fd8 <+8>:  movq   (%rsp), %rdi
    0x104cb1fdc <+12>: movabsq $0x10079e3c0, %r11
    0x104cb1fe6 <+22>: callq  *%r11
    0x104cb1fe9 <+25>: movq   %rax, %rdi
    0x104cb1fec <+28>: xorl   %esi, %esi
    0x104cb1fee <+30>: movabsq $0x1007e844e, %r11
    0x104cb1ff8 <+40>: callq  *%r11
    0x104cb1ffb <+43>: movq   %rax, 0x8(%rsp)
    0x104cb2000 <+48>: movq   (%rsp), %rdi
    0x104cb2004 <+52>: movabsq $0x10079e380, %r11
    0x104cb200e <+62>: callq  *%r11
    0x104cb2011 <+65>: movq   0x8(%rsp), %rcx
    0x104cb2016 <+70>: movq   %rcx, (%rax)
    <b>0x104cb2019 <+73>: addq   $0x18, %rsp</b>
    0x104cb201d <+77>: retq
 </pre>
 </td><td>
 <pre>
 Mono`System.EmptyArray`1<T_REF>:.cctor ():
    0x100fdf660 <+0>:  pushq  %rbp
    <b>0x100fdf661 <+1>:  movq   %rsp, %rbp
    0x100fdf664 <+4>:  subq   $0x10, %rsp</b>
    0x100fdf668 <+8>:  movq   %r10, -0x8(%rbp)
    0x100fdf66c <+12>: movq   -0x8(%rbp), %rdi
    0x100fdf670 <+16>: movabsq $0x10079e3c0, %r11
    0x100fdf67a <+26>: callq  *%r11
    0x100fdf67d <+29>: movq   %rax, %rdi
    0x100fdf680 <+32>: xorl   %esi, %esi
    0x100fdf682 <+34>: movabsq $0x1007e93ce, %r11
    0x100fdf68c <+44>: callq  *%r11
    0x100fdf68f <+47>: movq   %rax, -0x10(%rbp)
    0x100fdf693 <+51>: movq   -0x8(%rbp), %rdi
    0x100fdf697 <+55>: movabsq $0x10079e380, %r11
    0x100fdf6a1 <+65>: callq  *%r11
    0x100fdf6a4 <+68>: movq   -0x10(%rbp), %rcx
    0x100fdf6a8 <+72>: movq   %rcx, (%rax)    
    <b>0x100fdf6ab <+75>: leave</b>
    0x100fdf6ac <+76>: retq
</pre>
</td></table> 
