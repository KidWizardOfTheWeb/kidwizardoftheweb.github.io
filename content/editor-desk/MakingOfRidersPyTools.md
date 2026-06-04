+++
date = '2026-05-28T18:15:38-04:00'
draft = false
title = 'Making Of RidersPyTools'
+++

{{< toc >}}

## RidersPyTools - Python binding framework for Dolphin games

Link to project: https://github.com/KidWizardOfTheWeb/RidersPyTools

It has always been a dream of mine to get more people into programming and teach it as well. Certain languages and tools have made this easier, especially the flexibility of Python. I tend to use it for a lot of my current projects due to how easy it is to start up and get going, or to make a simple automation easier to do. 

When I realized how much it actually benefitted my reverse engineering toolkits, I thought of a way to hook the games I emulate and read/write data from them in realtime. This became possible when I discovered [py-dolphin-memory-engine](https://github.com/KidWizardOfTheWeb/py-dolphin-memory-engine), a package designed to hook into the dolphin process and do exactly that.

Using the power of inheritance and Python classes: **RidersPyTools** was created.

The goal of this project is simple: Allow a user using python code to modify values in memory as simply as if they were working with a decompilation. As long as a user has memory addresses, they could modify the package and add data structures as they please to fit their game. 

While created for Sonic Riders to serve as a bridge into learning the C++ codebase for Sonic Riders: Tournament Edition, any other game that dolphin can run can fork this repository and retool the structures to serve their needs. All of the power is provided by the two main structures of `GenericData` and `OffsetAttr`, which support any other datatype that's thrown at them. Feel free to use the repository for your game, or replace the dolphin memory engine dependency with another entirely.

For now, let's get into how this package works and what you can do with it.

### The problem: How Dolphin-Memory-Engine works
It is a simple package with commands that look like this:

```py
import dolphin_memory_engine as DME

# Hook to existing instance or silently throw RuntimeError
DME.hook()

addr_to_read = 0x80000000

# Read this address's value from memory, 8-bit int 
DME.read_byte(addr_to_read)

# Write an 8-bit int to this address in memory
DME.write_byte(addr_to_read, 0x1)
```

Simple as it may be, it does get quite tedious to write whether one wants to read or write a given value and what address that value must be at every time. Tons of read and write commands will just end up looking like alphabet soup, defeating the purpose of easy bindings and being unable to scale easily. Instead, I decided to utilize Python classes and overriding dunder methods to create a new approach.

This problem is easily solvable by breaking everything down into it's main components: representation and behavior.

### The solution: creating the memory attribute framework

#### Representation of data
For representation, a piece of memory for a game has an **address** (the memory location where it's located) and a **datatype** (the actual length of data of a variable to capture it's real value). Since we know this to be true, we can create a class to handle this information:

```py
# Most up to date version is located at: https://github.com/KidWizardOfTheWeb/RidersPyTools/blob/master/src/RidersPyTools_KC/include/OffsetAttr.py
"""
This is where a class is defined for each offset, containing the following:
1. Offset
2. Datatype
3. Struct type (if ptr)
"""

class OffsetAttr:
    def __init__(self, offset, datatype, ptr=None, struct=None):
        # REQUIRED TO HAVE BOTH OF THESE
        super().__setattr__('_offset', offset)
        super().__setattr__('_datatype', datatype)
        # Treat this class as a superclass to any other structs that may exist

        # If the attribute is a pointer, set the value it points to here
        if ptr is not None:
            # TODO: Add support for checking where the pointed address leads to
            super().__setattr__('_ptr', ptr)
            # self.__dict__["_ptr"] = ptr
            # self._ptr = ptr
        else:
            pass
```

We treat this class as a superclass to any memory of importance in the game. From generic single memory addresses to pointer structures or arrays, any attribute of the child class will store this data when `OffsetAttr` is used as a superclass. To note: these are set with `_` as a prefix, meaning offset and datatype are stored as a protected method and cannot be normally accessed by a user unless specifically implied.

Here's a pared down example of a possible structure you could create as a result:
```py
"""
Cut down example of: https://github.com/KidWizardOfTheWeb/RidersPyTools/blob/master/src/RidersPyTools_KC/Player.py
"""
import dolphin_memory_engine as DME

# All of these classes use OffsetAttr as a superclass. 
# Their parameters include the data necessary to fill in the superclass information
from .include.Controller import Controller
from .include.GenericData import GenericData
from .include.GearStats import GearStats
from .include.Constants import *

class Player:
    # Get/Set attribute functions go here, cut for the sake of space
    def __init__(self, playerNum, playerPtr=None):
        self.playerPtr = playerPtr + (0x1080 * playerNum)
        
        # The input ptr is always at the start of the playerPtr struct, so just read word from here and pass the Controller class to read all struct data as attributes
        ptr_start_addr = self.playerPtr

        # Go to pointer using DME's function for following pointers for controls, starts at offset 0
        self.input = Controller(DME.follow_pointers(ptr_start_addr, [0]), ptr)
        
        # Float data
        self.x = GenericData(ptr_start_addr + 0x1E4, f32)
        self.y = GenericData(ptr_start_addr + 0x1E8, f32)
        self.z = GenericData(ptr_start_addr + 0x1EC, f32)

        # GearStats[3] -> index per level
        # This is an array of structs of type GearStats
        self.gearStats = [GearStats(ptr_start_addr + 0x8DC, u32), GearStats(ptr_start_addr + 0x914, u32), GearStats(ptr_start_addr + 0x94C, u32)]

        self.currentAir = GenericData(ptr_start_addr + 0x984, u32)
        self.speed = GenericData(ptr_start_addr + 0xAAC, f32)
        self.speedAsInt = GenericData(ptr_start_addr + 0xABC, u32)

        self.rings = GenericData(ptr_start_addr + 0xB98, u32)

        # This is an array of bytes:
        # 1st byte = ms, 2nd byte = sec, 3rd byte = min
        # Note: on lap being completed, this resets to zero in-game.
        self.finishTime = [GenericData(ptr_start_addr + 0xFF8, u8), GenericData(ptr_start_addr + 0xFF9, u8), GenericData(ptr_start_addr + 0xFFA, u8)]

        self.currentLap = GenericData(ptr_start_addr + 0x102A, u8)
        self.level = GenericData(ptr_start_addr + 0x102E, u8)

        self.state = GenericData(ptr_start_addr + 0x1034, u8)
```

Single data addresses can be defined using `GenericData`, while specialized structs like `Controller` and `GearStats` have their own attributes that are linked here.

By creating classes with this setup, we can then build our dunder methods to do the real magic.

#### The behavior of the memory attribute framework

Tl;DR: a dunder method in Python is a method that's implicitly called when something like loading/storing/math operations/comparisons are performed on a variable (the same general operations types as assembly). The language allows you to override the behavior of these dunder methods when they're performed, like so:

```py
# Init ficticious class instance we made up
new_var = Class1()

# Set the attribute
new_var.attribute1 = new_value

# This calls __setattr__

# Class implementation could be:
def __setattr__(self, name, value):
    if value < 0:
        print("Negative value! We won't store this")
        return
```

You can perform any operations you want in a dunder method, Including using Dolphin memory engine.

Let's say I wanted to write to a value in memory. The regular way with DME (abbreviating Dolphin memory engine to DME for the rest of this post) is like this:

```py
DME.write_byte(addr, value)
```

If I wanted to write a value in Python, it's just like this:

```py
variable_to_write = new_value
```

Using dunder methods, we can combine the best of both worlds **at the same time!** Here's a bit about how setting a value works implicitly:
```py
    # Implemented in the class structure itself
    def __setattr__(self, name, value):
        global INIT_STATE

        # On startup, this allows everything to be assigned to the player object.
        # We NEED this for init ONLY.
        # Once every struct variable is done, SET INIT_STATE = False.
        # Once that is done, this will retrieve data from DME instead of setting the attribute's value.
        if INIT_STATE:
            super().__setattr__(name, value)
            return
        # Use this function for GenericData value assignment that isn't setting equal to a new object instance (that's handled in their own classes)

        # This gets our types and offsets (users cannot get these, they are protected from frontend).
        # check_DME_value = vars(self.__getattribute__(name))
        type_to_write = vars(self.__getattribute__(name))["_datatype"]
        offset_to_write = vars(self.__getattribute__(name))["_offset"]

        # Check our types, read from game, return value if found
        try:
            if type_to_write == u8 or type_to_write == s8 or type_to_write == Bool:
                DME.write_byte(offset_to_write, value)
            if type_to_write == u16 or type_to_write == s16:
                DME.write_byte(offset_to_write, value)
            if type_to_write == u32 or type_to_write == s32 or type_to_write == vu32:
                DME.write_word(offset_to_write, value)
            if type_to_write == f32:
                DME.write_float(offset_to_write, value)
        except RuntimeError as e:
            print("RuntimeError: DME is " + str(e) + ". Failed to write new value.")
        return    
```

While this seems like a lot going on for just setting a value, to a user using this package to program, it looks like this instead:

```py
# Assume player1 is of the Player class and rings is a generic attribute:
player1.rings = 100

# This set the rings memory address to a value of 100.
```

Since rings is defined earlier as a `u32` with a given address and as a `GenericData` type, the dunder method recognizes the type of the attribute as `u32` and gets the address/offset to write to from the `OffsetAttr` superclass, finally using DME to write the value implicitly.

We have successfully bound an attribute of the player class and set the value using only python code and implicit DME calls!

### What this allows for developers
For developers who want to do specific actions or behaviors in their games without having to compile a new build or write new assembly as geckos/AR codes, this could be a solution. There are some minor caveats to keep in mind however, as this framework does not provide a full replacement to programming games in general.

1. CPU speed. Depending on your processor, the implicit DME call might take a bit longer than expected. If you were to get the current value of an active timer in the game you're reading, it could change depending on how fast DME ends up reading. This might be mitigated with threading or some other form of delegation, but in something like a `while True:` loop, this could be problematic for things that must happen on an exact frame.

2. No online compatibility (at least with dolphin). Dolphin records any memory changes/program counter updates, packs it all, and sends it to the others in the room. When an outside source modifies the data in the game, dolphin does not account for this and throws a desync error. Therefore, any sort of cheating online with memory modification is already blocked due to causing instant desyncs. Memory reads online are entirely fine, as it does not change anything about execution or block it.

For me personally, these aren't huge issues to worry about. As long as your read/written values don't require pinpoint timing, using it for event signalling to external sources or writing new data is fine. If there were a way to pause game execution for a split second, possibly this issue could be corrected, but that's probably not going to happen anytime soon.

### Release and conclusion
During the initial release, I've created a hackathon revolving around RidersPyTools in the community that I currently develop for mainly that'll run for the month of June. I've already seen one fork that made a fullstack app that modifies in-game statistics in real time for testing and stores time trial data to a csv, so who knows what else people will make. Personally it beats debugging some simple stuff since everything is already predefined by me, and even if the dol file shifts addresses, I can drop in new addresses myself. 

I moreso hope that people may adapt it to their own dolphin emulated games for any extra things they can come up with. I have a few ideas personally, but I think it would be super helpful with streaming overlays that need data.

As usual, you can message me on discord at `kid.chameleon` or leave an issue on the repo for feature requests. Link to the repository is here: https://github.com/KidWizardOfTheWeb/RidersPyTools