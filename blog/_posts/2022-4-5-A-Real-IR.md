---
layout: post
title: "Compiler Reworks"
date: 2022-03-22 11:00:00 -0500
categories: ""
---

## Cleaning Up After Last Episode
In the previous update, I detailed how the entire middle and back end of the compiler fell apart. I had managed to get pointers working, but doing so exposed all the weaknesses in my naive first implementations of everything. Since then I have fixed up the IR to a much more capable form, reworked the entire code generator, and implemented proper register allocation.

## A Real IR
Previously, I had taken the lazy approach of treating the IR as a single linear block of three-address code. This worked fine for the simplistic register allocator I threw together, but with no notion of branching it wasn't very powerful. The main thing it lacked was the concept of lifetime differences within different branches of code. The new IR is a much more traditional system, where each function is linearized into a sequence of "basic blocks", each of which contains some code. It differs from a typical basic block implementation in one critical way though - normally a basic block is a section of straight-line code. This would mean that control flows linearly from the start of a block to the end. In my representation, it made more sense to allow branching within blocks, but jumps and calls will only ever target the beginning of a basic block.

## Linear Scan Register Allocation
The need for (and subsequent implementation of) the register allocator went hand-in-hand with the IR improvements. In order to get the backend of the compiler up and running, the lazy first-come-first-serve allocator with no spilling capabilities had to go. In its place is an implementation of linear scan allocation, with similar state save and restore functionality as before. The allocator itself isn't anything too special, but it took a lot of machinery under the hood to get it to work properly. The pseudocode in the paper I referenced last article really doesn't consider much by way of details, so a lot of the development time was spent simply trying and tweaking many iterations of supporting code. Now that the allocator properly supports spilling variables, the stack-based state tracking system I've been using for branch control has been beefed up a lot too. Now that variables can actually be spilled, it has a lot more work to do in terms of shuffling values around to make sure state remains consistent between different branch exits.

## Code Generation
After reworking the IR and register allocator, putting them to use in the code generator was much easier than before. Overall it's now a lot easier to see what's going on and to tweak and build functionality on top of what's already there. It's much less of a mess than how I was doing it before, and adding new features should be much more elegant to build up on top of what's already there.

## Show me Some Results!
Admittedly, a lot of what's discussed in this post is pretty hand-wavy and tough to get a mental model of just from reading. I wanted to throw together some diagrams to include, but it's already been so long since last post that I decided it'd just be better to get this out there. Now that the whole compilation pipeline is really starting to gel together, I hope to broaden my focus a bit. There are still a lot of big features on the horizon for the compiler, but I want to branch out into some visualization and profiling tools. 

| TAC Index |  |  |
|-------|-------|-------|
| - | Basic Block 1 | | 
| 0 | cmp i 2 | |
| 1 | jge basicblock 2 | Basic Block 2 |
| 2 | result = i | .t2 = i - 2 |
| 3 | jmp basicblock 1 | push .t2 |
| 4 | | .t1 = call fib |
| 5 | | .t4 = i - 1 |
| 6 | | push .t4 |
| 7 | | .t3 = call fib |
| 8 | | result = .t1 + .t3 |
| 9 | Basic Block 1| jmp basicblock 1 |
| 10| .RETVAL = result | |
| 11| ret .RETVAL  | |

```
Index
     | Basic Block 0    |
 0:  | cmp i 2          |
 1:  | jge basicblock 2 | Basic Block 2
 2:  | result = i       | .t2 = i - 2
 3:  | jmp basicblock 1 | push .t2
 4:  |       ----       | .t1 = call fib
 5:  |       ----       | .t4 = i - 1
 6:  |       ----       | push .t4
 7:  |       ----       | .t3 = call fib    
 8:  |       ----       | result = .t1 + .t3
 9:  | Basic Block 1    | jmp basicblock 1
10:  | .RETVAL = result |    
11:  | ret .RETVAL      |
```