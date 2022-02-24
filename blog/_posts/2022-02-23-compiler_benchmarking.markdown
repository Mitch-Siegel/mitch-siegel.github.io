---
layout: post
title: "Compiler Benchmarking and Imminent Features"
date: 2022-02-23 7:30:00 -0500
categories: ""
---

## Compiler Progress
Since the last post, I've upgraded every stage of the compilation pipeline to support the things necessary for flow control. At this point, if/else blocks and while loops are up and running almost perfectly. The main issue of keeping track of register states actually didn't end up being quite as difficult as I expected it to be. 

## Flow Control and Register State Tracking
Imagine some function which performs straight-line operations, then evaluates an if/else statement, and then performs some more straight-line operations afterwards. Before the if statement, the state of the registers is easily tracked since we know exactly what operations will happen. But what if the contents of the if block and the else block vary enough that register contents get all switched around? After we exit whichever branch we ended up taking, the registers could be in one of two states. This is no good to continue executing code - how do we know which registers contain what?

My solution to this issue was to implement a sort of "state snapshot" system. Before any branching operation is done, the code generator copies the current register states (register contents, and which registers are live) and pushes the duplicated states to a stack. Then, whether it's generating code for an if statement or the matching else, it can refer to the most recent state snapshot. This allows the code generator to always start from a known state. Similarly, once code generation for an if or else block is done, there's a reference for what the machine state should look like on exit from these branches. 

Using the snapshot system, the states of the registers can be restored to what they were before a branch was evaluated. Then, when code has been generated for all the branch conditions, the most recent state snapshot is popped off the stack. Now the code generator can move on as if nothing happened. This stack-based approach naturally allows for easy nesting of branching operations too.

This same concept applies to the while loop - whether or not the loop condition is met, the starting state is always known. Because of this, the state can be restored to consistency regardless of the outcome of the branch. Using this system, I should be able to put together a for loop and switch statement without too much extra hassle.

## Turing Complete, but Not Very Dangerous
Although the compiler still needs some manual help to assemble, call, and run its output, it's now at a point where it's nearing usability to do some interesting stuff. That said, without any way to access memory directly, actually pulling these things off is more trouble than it's worth. My next major step will be to implement pointers and pointer arithmetic. These will still follow the same typeless, 16 bit unsigned arithmetic I've been using so far. From there I hope to move along quickly to arrays, and allow for local array allocation within functions too. 

Pointers alone will make the language properly usable, but I think arrays (and the syntactic sugar they provide on top of pointer arithmetic) will really bring things together and add a real edge to it.

## Compiler Benchmarking (The Bad, the Worse, and the Recursive)
So far, most of my testing has been on silly and meaningless examples just to verify and troubleshoot features. With flow control, I've been able to use the Fibonacci sequence and finding primes to test things out. The Fibonacci algorithm I'm using is the simplest possible (and quite poorly performing) recursive algorithm. The prime finder is an atrociously naive implementation which iteratively tries to divide all numbers less than its input to ascertain primality.

### Fibonacci Numbers
Given the 16 bit word size of the machine, the first 20 numbers in the series seems like a nice round number to benchmark. The 25th entry is just too big at 75,025, so 20 will be the number I'll use.

### Primes
For the primes benchmark, I decided to find all prime numbers under 1,000, or the first 168 primes. Above 3,000 or so the emulator really starts to chug doing all the iterative division for each number. Maybe 10,000 will be a target for the future, when I can implement a more efficient algorithm.

### Benchmark Results

| Benchmark | Instructions Executed |
|-------|-------|
| First 20 Fibonacci numbers: | 859,483 |
| Just the 20th Fibonacci number: | 328,363 |
| All primes under 1000: | 6,923,578 |

There's not too much to say about the primes for an implementation this naive, but it's interesting to see how the exponential recursion really catches up with the Fibonacci series. The 20th entry alone took almost half as many instructions as it took to calculate as the entire series!

It has been really cool to see the source code for these algorithms get compiled and run, but their performance is absolutely woeful. With only functions, local variables, if/else's, and while loops, what can you really do though? I hope to continually reimplement and benchmark these algorithms as I add features to the language and improve the code generation. As more features come along, I will also be able to add more interesting benchmarks.

Below are the exact source code and assembly outputs for each of the examples. There is some inline assembly used to (hackily) output the calculated values. Even without much analysis, it's pretty easy to see that the code generation could be better. I'm looking forward to drastically improving these benchmark performances using pointers in the near future!

#### Fibonacci Source Code
```
# this function computes the nth fibonacci number 
fun fib(var n)
{
    if(n < 2)
        return n;
    else
        return fib(n - 1) + fib(n - 2); 
}

# find all fibonacci numbers up to and including n
fun firstnfibs(var n)
{
    var i = 0;
    while(i < n)
    {
        fib(i + 1);
        asm
        {
            out %r0
        };
        i = i + 1;
    }
}
```

#### Primes Source Code
```
# software modulo! (are you having fun yet?)
fun modulo(var a, var b)
{
    while(a >= b)
    {
        a = a - b;
    }
    return a;
}

# return whether or not a number is prime
fun isPrime(var n)
{
    var i = 2;
    while(i < n)
    {
        var result = modulo(n, i);
        if(result == 0)
            return 0;

        i = i + 1;
    }
    return 1;
}

fun firstNPrimes(var n)
{
    var i = 2;
    var foundNum = 0;
    while(foundNum < n)
    {
        var result = isPrime(i);
        if(result == 1)
        {
            asm
            {
                out %r2
            };
            foundNum = foundNum + 1;
        }
        i = i + 1;
    }
}
```

#### Fibonacci Compiler Output
```
fib:
	sub %sp, $0
	push %r1
	mov %r0, 4(%bp)
	cmp %r0, $2
	jge fib_1
	mov %r0, 4(%bp)
	jmp fib_done
fib_1:
	mov %r0, 4(%bp)
	sub %r0, $2
	push %r0
	call fib
	mov %r1, 4(%bp)
	sub %r1, $1
	push %r1
	mov %r1, %r0
	call fib
	add %r1, %r0
	mov %r0, %r1
	jmp fib_done
fib_done:
	pop %r1
	add %sp, $0
	ret 1
firstnfibs:
	sub %sp, $2
	push %r1
	push %r2
	mov %r0, $0
firstnfibs_1:
	cmp %r0, 4(%bp)
	jge firstnfibs_2
	mov %r1, %r0
	add %r1, $1
	push %r1
	mov %r2, %r0
	call fib
	out %r0
	add %r2, $1
	mov %r0, %r2
	jmp firstnfibs_1
firstnfibs_2:
firstnfibs_done:
	pop %r2
	pop %r1
	add %sp, $2
	ret 1
```


#### Primes Compiler Output
```
modulo:
	sub %sp, $0
modulo_1:
	mov %r0, 4(%bp)
	cmp %r0, 6(%bp)
	jl modulo_2
	mov %r0, 4(%bp)
	sub %r0, 6(%bp)
	mov 4(%bp), %r0
	jmp modulo_1
modulo_2:
	mov %r0, 4(%bp)
	jmp modulo_done
modulo_done:
	add %sp, $0
	ret 2
isPrime:
	sub %sp, $4
	push %r1
	push %r2
	mov %r0, $2
isPrime_1:
	cmp %r0, 4(%bp)
	jge isPrime_2
	push %r0
	push 4(%bp)
	mov %r1, %r0
	call modulo
	cmp %r0, $0
	jne isPrime_3
	mov %r2, $0
	mov %r0, %r2
	jmp isPrime_done
isPrime_3:
	add %r1, $1
	mov %r0, %r1
	jmp isPrime_1
isPrime_2:
	mov %r0, $1
	jmp isPrime_done
isPrime_done:
	pop %r2
	pop %r1
	add %sp, $4
	ret 1
firstNPrimes:
	sub %sp, $6
	push %r1
	push %r2
	mov %r0, $2
	mov %r1, $0
firstNPrimes_1:
	cmp %r1, 4(%bp)
	jge firstNPrimes_2
	push %r0
	mov %r2, %r0
	call isPrime
	cmp %r0, $1
	jne firstNPrimes_3
	out %r2
	add %r1, $1
firstNPrimes_3:
	add %r2, $1
	mov %r0, -2(%bp)
	mov %r0, %r2
	jmp firstNPrimes_1
firstNPrimes_2:
firstNPrimes_done:
	pop %r2
	pop %r1
	add %sp, $6
	ret 1
```