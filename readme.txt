To handle the memory allocation requests, a new memory allocator was written. 

How it works:

The 128KB RAM is divided into 2 "virtual" 64KB banks: the lower and the upper bank. A port selects which "virtual" 64KB RAM bank is mapped to the physical 64KB RAM.

In order to execute code that switches from the lower to the upper bank (and back...), that piece of code must be present at the same physical address in both of banks (the so called "shadow code)". It is impossible to store shadow code in the upper bank using only programs resident in RAM, for this only ROM-based code may help; my version of SCM contains an API function ( $2B ) which moves a byte to a given address of the upper RAM bank. This API call is not documented (it cannot be found in the SCM manual), I've found-it by browsing through the SCM source code... 

Therefore, by using this $2B SCM API call, the shadow code can be moved to the upper RAM bank.

But, for the memory allocator, we have also another important constraint: the shadow code must be stored in top of the physical 64KB RAM, in order to offer as much possible RAM available to be allocated (storing at the bottom of the physical 64KB RAM is out of question, we have there the 100H BIOS buffers area). There is only one possible solution: storing a small shadow code on top of CP/M's BIOS. My shadow code is less than 40H. 

Now, studying the "classic" RC2014 CP/M that is currently used with 64/128MB CF's, I discovered that the BIOS "eats" practically all the space to FFFF ! There was no room for my shadow code! 

But, I remembered that I built recently a "custom" CP/M for RC2014 systems that use a 64MB CF. This version of CP/M loads its BDOS at DA00, and has also the advantage of having a small "free" RAM area on top of the BIOS (around 40H ! ). This was a perfect match, I stored there (FFA0 - FFFF) the shadow code. This allows also having (almost) 64KB RAM available to be allocated in the upper RAM bank! 

If the usual call to malloc fails, the new allocator accesses the upper 64KB RAM bank and allocates there the buffer. 

Besides the address of the buffer, a byte containing the RAM bank index is returned too (0=lower 64KB RAM, 1=upper 64KB RAM).

To access a buffer reserved in the upper 64KB RAM bank, special functions must be used to read/write a byte, a word, or a string.
 
Text files up to 70KB can be edited. 

But, this comes with a cost: when opening a file to be edited, the editor reads all the text and store this text in the dynamic memory, and this takes time, because every single byte that has to be moved to the upper RAM bank is not simply moved, as in case of the lower bank, but is passed to the shadow code in order to be stored in the upper RAM bank. Also, a buffer for every new text line that is read must be allocated, and this allocation means also accessing the shadow code, in order to manage (read/write) the data structures that handle the available memory pool.  

Once the file is read, the browse/search operations prove to be quick (compared for example with the performance of WordStar), because no more file read operations are needed, all the text is in the memory.
