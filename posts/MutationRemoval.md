---
layout: default
title: "Removing x86 Mutation"
---

## Background

I recently started reversing a driver which makes heavy use of a certain unnamed commercial protector. One of the most frequently used features of that protector is mutation. Mutation basically inserts random garbage instructions between meaningful instructions, which makes analysis significantly harder. The functions I analyzed seemed to have some additional protections applied, which includes:

- Extended basic blocks
- Duplicated blocks
- Fake conditional jumps

For this writeup, I am focusing solely on the mutation.

## Analyzing the obfuscation

Here's an example of the obfuscation:

![image](/assets/images/mutation.png)

Right away, the following marked instructions stood out as junk:

```nasm
    rol     r9w, 9 // JUNK
    mov     [rsp+arg_8], rbx
    add     r9w, 0AE93h // JUNK
    ror     r9w, cl // JUNK
    mov     [rsp+arg_10], rbp
    inc     r9d // JUNK
    mov     [rsp+arg_18], rsi
    bt      bx, 0Dh // most likely junk, but no guarantees can be made without analyzing other blocks
    sal     r9w, cl // JUNK
    mov     r9, cs:qword_FFFFF80584BBBA00
    test    esi, 84596310h // JUNK
    stc  // JUNK
    mov     r8b, 1
    cmp     dx, 0F618h
    jle     loc_FFFFF80584C8F35C
```

The mutation follows a pretty obvious pattern. They will only insert junk instructions when they know the result will be overwritten. Once I saw how simple the mutation was, it became a matter of creating an algorithm to identify mutation.

## Considerations

Before I started writing my own solution, I looked into existing solutions & tooling. I was only able to find one existing writeup detailing mutation removal, which I highly reccomend reading [here](https://www.usualsuspect.re/article/automatic-removal-of-junk-instructions-through-state-tracking). The author of the linked writeup used Triton, which didn't feel like the right option in my case. In hindsight, I probably should've used Triton(or another existing binary analysis framework), but there were some caveats:

- I had no use for alot of Triton's more advanced features in this case
- I wasn't really sure how handling some of the other obfuscation I ran into would go with Triton
- I felt like writing my own tool would offer more flexibility
- I planned to use this tool for some additional uses that I will elaborate on at some point

None of the existing binary analysis frameworks seemed like the best option, so I just decided to reinvent the wheel and write my own.

## Getting Started

Once I got my initial tooling setup, I was finally able to start working on removing the mutation. It was surprisingly more difficult than I thought. My first attempt went like this:

1. Grab a basic block
2. Iterate through every instruction in the basic block, and retrieve the index of the first instruction who wrote to the result, along with the first instruction who read the result. (result is referring to any of the registers written to by a given instruction)
3. Discard the instruction if the result is overwritten before it is read

It somehow worked surprisingly well. However, it just wasn't accurate enough. Thinking back on it, there are some glaring issues. Take the following made up snippet as an example

```nasm
    rol     r9w, 9
    add     r9w, 0AE93h
    ror     r9w, cl
    inc     r9d
    sal     r9w, cl
    mov     r9, cs:qword_FFFFF80584BBBA00
```

If you determine instruction validity based off whether or not the result is read before it is written, it is going to fail here(unless you don't count ReadWrite instructions as writes, which would just result in messed up output alot of the time). This problem bothered me for a week or two, but something just clicked eventually. Going back to the obfuscated snippet from above:

```nasm
    rol     r9w, 9 // guaranteed junk
    mov     [rsp+arg_8], rbx
    add     r9w, 0AE93h // guaranteed junk
    ror     r9w, cl // guaranteed junk
    mov     [rsp+arg_10], rbp
    inc     r9d // guaranteed junk
    mov     [rsp+arg_18], rsi
    bt      bx, 0Dh
    sal     r9w, cl // guaranteed junk
    mov     r9, cs:qword_FFFFF80584BBBA00 // legit instruction
    test    esi, 84596310h
    stc
    mov     r8b, 1
    cmp     dx, 0F618h
    jle     loc_FFFFF80584C8F35C
```

When a human reads this, they instantly know that the first few instructions which write to R9 are junk. The process most of us go through to identify junk can sortof be broken down into this:

- Look at all read/writes to R9
- Find the last write to R9
- Trace backwards from that last write and check if R9 is actually used before it is overwritten.

After some more thinking, I decided to try implementing a sortof dependency graph which keeps track of which instructions depend on each other. Here is the result of the dependency graph(**Note: I reversed the order of the dependency graph to make analysis easier**):

```nasm
0: jle near ptr 0FFFFF80584C8F35Ch - Read - Write
1: cmp dx,0F618h - Read - Write
2: mov r8b,1 - Read - Write
3: stc - Read - Write
4: test esi,84596310h - Read - Write
5: mov r9,[0FFFFF80584BBBA00h] - Read - Write
6: sal r9w,cl - Read - Writes: 5
7: bt bx,0Dh - Read - Write
8: mov [rsp+20h],rsi - Read - Write
9: inc r9d - Reads: 6 - Writes: 6 : 5
10: mov [rsp+18h],rbp - Read - Write
12: add r9w,0AE93h - Reads: 11 : 9 : 6 - Writes: 11 : 9 : 6 : 5
13: mov [rsp+10h],rbx - Read - Write
14: rol r9w,9 - Reads: 12 : 11 : 9 : 6 - Writes: 12 : 11 : 9 : 6 : 5
```

The output isn't the most intuitive, so I will try to explain. The numbers followed by "Reads:" are the indexes of the instructions which read the result of that instruction. The numbers followed by "Writes:" are the indexes of the instructions which write to the result of that instruction. Before I get to the code, I want to manually explain the logic that I used to identify junk instructions.

- `5: mov r9,[0FFFFF80584BBBA00h]` - Writes to r9, and has 0 instructions that read/write the result. Because of this, we can assume that it is a good instruction - KEPT
- `6: sal r9w,cl` - Writes to r9, which is overwritten before it is read - DISCARDED
- `9: inc r9d` - Writes to r9, which is overwritten before it is read(keep in mind that we discarded instruction #6, so we can guarantee that it is overwritten before it is read) - DISCARDED
- `12: add r9w,0AE93h` - Same as #9. Result is overwritten before it is read - DISCARDED
- `14: rol r9w,9` - Same as #9. Result is overwritten before it is read - DISCARDED

You're probably wondering about register sizes at this point. In my case, I was able to get away with just converting each register to it's highest form(i.e r9w just gets converted to r9).

You're also probably wondering about those junk eflags writes. I'm not covering those indepth because the solution isn't really complicated. There are better ways to do it, but my algo looks like this:

1. Identify all instructions whose sole purpose is to modify eflags(stc, cmp, clc, cmc, ect)
2. Remove the instruction if the containing block's exit instruction does not read it

## Implementing the algorithm

First I needed a structure to store the dependency graph:

```csharp
    public class InstructionDependency
    {
        /// <summary>
        /// Source instruction
        /// </summary>
        public Instruction instruction;

        // Dictionary of the indexes of instructions which read the result of the sourceInstruction
        public Dictionary<int, Instruction> resultReaders = new Dictionary<int, Instruction>();

        // you get the rest
        public Dictionary<int, Instruction> resultWriters = new Dictionary<int, Instruction>();
        public Dictionary<int, Instruction> flagReaders = new Dictionary<int, Instruction>();
        public Dictionary<int, Instruction> flagWriters = new Dictionary<int, Instruction>();
    }

    public class BlockDependencyGraph
    {
	    public List<InstructionDependency> dependencies = new List<InstructionDependency>();
    }
```

Then I build the dependency graph:

```csharp
    /// <summary>
    /// Builds a dependency graph for a basic block
    /// </summary>
    /// <param name="block"></param>
    /// <returns></returns>
    private BlockDependencyGraph BuildDependencyGraph(BasicBlock block)
    {
        BlockDependencyGraph dependencyGraph = new BlockDependencyGraph();
        foreach(var instruction in block.instructions)
        {
            InstructionDependency dependency = new InstructionDependency();
            dependency.instruction = instruction;

            // retrieve the registers and flags written to by the current instruction
            var srcIndex = block.instructions.IndexOf(instruction);
            var writtenRegisters = instruction.GetPossibleWrittenRegisters();
            var writtenFlags = instruction.GetPossibleWrittenFlags();

            // iterate through all instructions later in the block and store dependencies
            var nextInstructions = block.instructions.Skip(srcIndex + 1);
            foreach(var nextInstruction in nextInstructions)
            {
                // Calculate index of the dependency and reverse it since we iterate through the dependency graph backwards
                var reversedIndex = block.instructions.Count - 1 - block.instructions.IndexOf(nextInstruction);

                // store the instructions which read the result of the current instruction
                if(nextInstruction.GetPossibleReadRegisters().Intersect(writtenRegisters).Any())
                    dependency.resultReaders.Add(reversedIndex, nextInstruction);

                // store the instructions which write to the result of the current instructions
                if(nextInstruction.GetPossibleWrittenRegisters().Intersect(writtenRegisters).Any())
                    dependency.resultWriters.Add(reversedIndex, nextInstruction);

                // store the instructions which reads any flag modified by the current instruction
                if(nextInstruction.GetPossibleReadFlags().Intersect(writtenFlags).Any() && !dependency.flagReaders.Keys.Contains(reversedIndex))
                    dependency.flagReaders.Add(reversedIndex, nextInstruction);

                // store the instructions which write to any flag modified by the current instruction
                if(nextInstruction.GetPossibleWrittenFlags().Intersect(writtenFlags).Any() && !dependency.flagWriters.Keys.Contains(reversedIndex))
                    dependency.flagWriters.Add(reversedIndex, nextInstruction);
            }

            dependencyGraph.dependencies.Add(dependency);
        }

        return dependencyGraph;
    }
```

Finally, I analyze the dependency graph and remove mutation(this function is pretty lengthy):

```csharp
    public void RemoveBlockMutations(Node node)
    {
        // Retrieve the dependency graph and reverse it so that we can iterate backwards
        var depGraph = BuildDependencyGraph(node.GetBlock());
        var dependencies = depGraph.dependencies;
        dependencies.Reverse();
        LogDependencyGraphInfo(depGraph);

        // Grabs the first instruction whose flag result is read by the first instruction in the dependency graph
        // NOTE: The frst instruction in the dep graph will always be the exit instruction(jmp, ret, ect), because the dep graph is reversed
        var flagResult = dependencies.FirstOrDefault(x => x.flagReaders.Keys.Contains(0));

        // Remove junk eflag writes
        if (flagResult != null)
        {
            // Remove all instructions which modify flags without making any meaningful changes
            node.GetBlock().instructions.RemoveAll(x => x.IsOnlyFlagInstruction() && x != flagResult.instruction);
        }


        foreach(var insnDependency in dependencies)
        {
            var depIndex = dependencies.IndexOf(insnDependency);
            if (insnDependency.instruction.IsOnlyFlagInstruction())
            {
                // TODO: More eflags analysis
            }

            else
            {
                bool isWritten = insnDependency.resultWriters.Values.Any();
                if (isWritten)
                {
                    bool isRead = insnDependency.resultReaders.Values.Any();
                    if (isRead)
                    {
                        // discard the instruction if it is overwritten before it is read
                        int firstRead = insnDependency.resultReaders.Keys.Max();
                        int firstWrite = insnDependency.resultWriters.Keys.Max();
                        if (firstWrite > firstRead)
                        {
                            // Remove the instruction from the containing block
                            node.GetBlock().instructions.Remove(insnDependency.instruction);

                            // remove all dependencies to the discarded instruction
                            foreach (var modifiedDep in dependencies)
                            {
                                modifiedDep.resultReaders.Remove(depIndex);
                                modifiedDep.resultWriters.Remove(depIndex);
                            }
                        }
                    }

                    // if an instruction is overwritten without being read, assume it is junk and discard it
                    else
                    {
                        // Remove the instruction from the containing block
                        node.GetBlock().instructions.Remove(insnDependency.instruction);

                        // remove all dependencies to the discarded instruction
                        foreach (var modifiedDep in dependencies)
                        {
                            modifiedDep.resultReaders.Remove(depIndex);
                            modifiedDep.resultWriters.Remove(depIndex);
                        }
                    }
                }
            }
        }
    }
```

That's really all there is. As of right now I can't post the entire source of my tool because I'm using it for analyzing a commercial application, but I may post a cleaned up version at some point. Here are a bunch of my [helper methods](https://pastebin.com/F80yR8WD) which should make the code alot easier to understand.

Some of the libraries I used include:

- [ICED](https://github.com/0xd4d/iced)
- [Rivers](https://github.com/Washi1337/Rivers)
- [AsmResolver](https://github.com/Washi1337/AsmResolver)

## Output

```nasm
    rol     r9w, 9 // JUNK
    mov     [rsp+arg_8], rbx
    add     r9w, 0AE93h // JUNK
    ror     r9w, cl // JUNK
    mov     [rsp+arg_10], rbp
    inc     r9d // JUNK
    mov     [rsp+arg_18], rsi
    bt      bx, 0Dh // most likely junk, but no guarantees can be made without analyzing other blocks
    sal     r9w, cl // JUNK
    mov     r9, cs:qword_FFFFF80584BBBA00
    test    esi, 84596310h // JUNK
    stc  // JUNK
    mov     r8b, 1
    cmp     dx, 0F618h
    jle     loc_FFFFF80584C8F35C
```

The obfuscated basic block turns into:

```nasm
    mov [rsp+10h],rbx
    mov [rsp+18h],rbp
    mov [rsp+20h],rsi
    mov r9,[0FFFFF80584BBBA00h]
    mov r8b,1
    cmp dx,0F618h
    jle near ptr 0FFFFF80584C8F35Ch
```

[Full Obfuscated Function Output](https://pastebin.com/g44XhBKU)

[Full Deobfuscated Function Output](https://pastebin.com/D0s4rZQi)

Also note there may be some messed up conditional jumps in the deobfuscated output. There is a flaw somewhere in my optimizer which fails to replace a fake conditional jump with an unconditional jump. Aside from that, I haven't found anything wrong with the output.

## Conclusions

My deobfuscator isn't perfect. I didn't really care about performance(I see the 1000 performance issues but I will get to that at a later point). Performance aside, I'm happy with the result. So far I haven't been able to find any cases where I optimized out a valid instruction. Any suggestions/criticisms are welcome!
