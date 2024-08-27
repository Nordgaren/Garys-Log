# Does anyone know how to use pointers, anymore?
Inflammatory article about a serious topic. How did this happen? Why are we wasting cycles on this? How did the compiler
even allow this to happen?

## What is this about
So, I have been wanting to just document some of the reverse engineering work that has been done by fellow soulsmodders,
and myself. For one, so that we have a bit more documentation on the inner workings on the Dantelion 2 engine, the engine
that the souls games are on, and a lot of the Armored Core games, as well. Kings Field? IDK. Maybe, I haven't looked. I 
wanted to expand [ErdTools](https://github.com/Nordgaren/Erd-Tools/tree/main), A tool for debugging the game as it runs, 
so that it could read the player inventory,as well as delete items and move them between inventories, so, I had to go and 
track down the code that handles all of this. I also need this code for some future projects, but, for now, I figured it 
would be cool to get started documenting things by documenting how the player inventory works.  

I found something interesting, and looking into it just a bit deeper, it seemed to be an issue in a lot of places.

## The offending code
I don't really know how terrible this is, but, it can't be great on a CPU that relies on branch prediction for speed. Here
is the post from [Twitter](https://x.com/NotNordgaren/status/1827627823454171535) that I originally posted.

![problem.jpg](problem.png)

This picture is similar to the picture with the original question, but with more context. I have drawn arrows for the important
jumps in red, and I also pointed out the allocation we are initializing in blue. I also highlighted `R14`, which is where 
the base of our inventory resides, for our indexing math. At the top, we start by calling `AllocateAligned` on our main
heap allocator (Of which the setup is omitted, just trust me, bro). After the call we have our pointer returned in `RAX`, 
as this is the convention in x86. We then test RAX against itself. The [`TEST`](https://www.felixcloutier.com/x86/test) mnemonic
is an instruction that will and it's two operands together, and set various flag registers, one of which is `ZF`, or Zero 
Flag. This flag is 1, or true, if the resulting value is 0. This can only happen with a value of zero in both operands,
so this make a very efficient way of testing a pointer for null. We then move on to a [`JNZ`](https://www.felixcloutier.com/x86/jcc), which is our first jump. This 
instruction will look at `ZF` and jump if it's set to 1. If we don't jump here, we fall to an unconditional jump that exits
the function. This is good, and what we should expect from a C++ program like this.

What comes after is a bit peculiar, because, I know compilers are smarter than this. That first test has us jump and avoid
exiting the function, if we get a non-null pointer back. This part of the function immediately adds `0x10`, which is 16 bytes,
to the pointer value with a [`LEA`](https://www.felixcloutier.com/x86/lea), or `Load Effective Address` instruction. This 
instruction loads the address of something into a register, and is optimized for pointer math, so, is often used for traversing
structures, etc. This is where the start of the actual array that is going to be initialized and used for the inventory. We
store this in `R14`.

We keep going, do some indexing math, do a check, call a function, and then we check to see if we have reached the last
item in the array, which is in `R15`, and the current item index in `RDI`. We then jump right back to where we are doing
pointer math with `R14` and friends. You may ask, where is the issue? Well, I kinda skipped over the check we do after the
index math, because, that is the issue. Remember, `TEST` is going to and it's operands together and set the `ZF` flag to
one ONLY IF the number is ZERO, as, anding any other number with any other number will result in something other than zero.

There are two instructions here, but in the end we take the value that is in `R14` and add `RAX * 0x8` to it, and then store
that pointer value in `RCX`. Since the only operations we have done to `R14` are addition operations, one of which was adding
16 to whatever value was in `RAX` and storing it in `R14`, we will never have a situation where this math results in zero.
That means our `TEST RCX, RCX` will never fail. 

So, it's one instruction, right? Why is this an issue? Well, in this instance
it's actually not bad at all, as this code only gets called at the game startup, but, it is done in a loop that has to iterate
thousands of times, and, the loops never operate on a constant (In fact, this function makes three inventories, and each
ones size is calculated at the start of the function). So, this behaviour could be quite rough on branch prediction, and 
this game only runs on CPUs that heavily favor branch prediction. On top of this, if that earlier null check wasn't there,
this code WILL crash your system on a null pointer, as the null check will fail and `0x10` is still invalid memory (on most
machines, I guess).

Here is a look at the pseudo code in Ghidra, cleaned up a little bit. I added a `ShimmedInventoryItemData` so that Ghidra
would show a bit what I am talking about. `itemId` is just the first field in the `InventoryItemEntry`, which is why you
see the code getting the address of it (silly). Again, this is pseudo code, but, I honestly thought Ghidra was drunk, when
I first saw this.

![pseudocode.png](pseudocode.png)

Turns out, this is not the only place they do this. I went and started looking at the references for this allocation function,
of which Ghidra has detected 137 call sites.

![allocator-function.png](allocator-function.png)

In fact, as far as I can tell, every time they put this shim on their allocations, they end up making the same mistake. Here
is another example from Elden Ring. I have no idea where this code is, I just found that it calls this allocator function
and then make another condition that cannot fail.

![another-one.png](another-one.png)


So, is FromSoft adding to their pointers manually, or do they have some kind of generic shim structure that they use, and 
they just don't understand how pointers work? Could there be some kind of abstraction that gets inlined by the compiler,
that has this null check in it? That doesn't really make any sense. Regardless, the `TEST` instruction has to be generated
by them checking `if (&alloc->firstEntry == nullptr)`, which just isn't going to be null. You are essentially asking the
compiler to add an offset when you reference fields further the first one (ignore Ghidras stupid pseduo code above where
it's referencing the first field. It's not incorrect, if you wanted the base of that structure. It's just stupid to write
it like that).

## Conclusion
Okay, so, I explained my problem, but, do I have a solution? Theoretically. I think we could probably write some Ghidra scripts
to check for this kind of thing by looking at all of the calls to this allocator, or even just every test instruction, and 
follow the registers involved backwards, to figure out if there's every a non-zero value being added to before being tested
against itself. I think this could be a free win for those who are looking to make performance mods to make any of the souls
games. Starting here would be great, and I will post the results if/when I get around to making a script to patch these unnecessary
checks out. This might even be something that has been going on for a while, which I will update here shortly if I, or any
of the other soulsmodders can find this behaviour in previous games.  

## Glossary

Shim
: Term for a piece of memory at the start of an allocation that holds information about said allocation. Usually hidden
from the user of the memory by the allocator, as it's typically information for the allocator to use during cleanup.

Bingus
: Bald cat.
