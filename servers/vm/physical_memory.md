## Physical Memory

After minix kernel intialization, **vm** can get information about size of 
physical memory, available physical memory chunks from kernel. **vm** uses 
these data to initialize physical memory manager. **vm** keep track of unused
physical memory by means of memory page *bitmap*, the bitmap keep track `2**32 = 4Gb` 
memory (bitmap takes up `2 ** (32 - 12) bytes = 1Mb`, if `PAGE_SIZE = 4k`), although
true memory is less than `4Gb`.

**vm** uses `phys_clicks alloc_mem(phys_clicks clicks, u32_t falgs)` to handle physical memory 
requests. Physical memory requests may have some special requirements, 
such as **low physical memory address**, **memory alignment** and **big memory page**.

The reciprocal of above function is `void free_mem(phys_clicks base, phys_clicks clicks)` 
which free physical memory previously allocated by `alloc_mem()`.


### Init

Free page bitmap is initialzed by `void mem_init(struct memory* chunks)` function called from
**vm** initialization routine. This function firstly set all entries of the page bitmap used,
then frees memory according to `chunks` array, size of the array is hard code.

``` c
void mem_init(struct memory *chunks)
{
  int i, first = 0;
  total_pages = 0;

  memset(free_pages_bitmap, 0, sizeof(free_pages_bitmap));

  /* Use the chunks of physical memory to allocate holes. */
  for (i=NR_MEMS-1; i>=0; i--) {
  	if (chunks[i].size > 0) {
		phys_bytes from = CLICK2ABS(chunks[i].base),
			to = CLICK2ABS(chunks[i].base+chunks[i].size)-1;
		if(first || from < mem_low) mem_low = from;
		if(first || to > mem_high) mem_high = to;
		free_mem(chunks[i].base, chunks[i].size);
		total_pages += chunks[i].size;
		first = 0;
	}
  }
}
```

