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
The RGCTX array initially has space for 4 pointers, MRGCTX has space for 6.
