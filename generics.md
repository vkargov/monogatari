## Lazy fetch.
Goes through `generic_trampoline_rgctx_lazy_fetch`.
For `Dictionary``2` it will likely be first called during its constructor execution:
```
   76           public Dictionary(int capacity, IEqualityComparer<TKey> comparer)
   77           {
   78               if (capacity < 0) throw new ArgumentOutOfRangeException(nameof(capacity), capacity, SR.ArgumentOutOfRange_NeedNonNegNum);
   79               if (capacity > 0) Initialize(capacity);
-> 80               this.comparer = comparer ?? EqualityComparer<TKey>.Default;
   81   #if !MONO
   82               if (this.comparer == EqualityComparer<string>.Default)
   83               {
   ...
```
The RGCTX array initially has space for 4 pointers, MRGCTX has space for 6. Relevant counters: "(M)RGCTX num arrays alloced", "(M)RGCTX bytes alloced".
It is assigned to `class_vtable->runtime_generic_context`.
The first slot int the array references the next array chunk, see the [generic sharing documentation](http://www.mono-project.com/docs/advanced/runtime/docs/generic-sharing/#mrgctx-lazy-fetch-trampoline).
