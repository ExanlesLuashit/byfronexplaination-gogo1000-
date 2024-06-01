Analyzing Byfron - Part 1: Starting Out​
Introduction​
Hello everyone. this is gocart98. I apologize for my recent inactivity, I've been quite busy preparing for this series.

I'm excited to announce that I'm launching a new multi-part series titled "Analyzing Byfron," where I'll walk you through the step-by-step process of reverse engineering Byfron.

A link to the next part will be posted under this thread once it's available, allowing you to easily pick up where you left off. Stay tuned for more in-depth explorations of Byfron's inner workings.

Credits​
Before we start, here is a list of this thread's authors.
Dr McDingle
stretchnuts

Table of contents​
Analyzing Byfron - Part 1: Starting Out
Introduction
Credits
Table of Contents
Overview
High-level View of Byfron
The Basics
Outline
Final Thoughts
Overview​
Byfron is typically referred to as an "anti-cheat," while this terminology is correct, you must understand the difference between Byfron's "AC" and traditional anti-cheats. Byfron is more accurately described as an "anti-tamper;" it differs from traditional anti-cheats. Traditional anti-cheats pivot towards cheat detection.

To protect their detection vectors and halt reverse engineers, typically, they obfuscate the code belonging to their anti-cheat.On the other hand, Byfron simply attempts to make reverse engineering of the game itself harder.It encrypts the game code and decrypts it only when needed (when execution occurs), and also protects the imports.

It aims to make reverse engineering harder in general. When Byfron was used in other games, it was accompanied by another anti-cheat. The goal of Byfron was to make reverse engineering of the game harder, as well as development on cheats. It isn't meant to work as a full anti-cheat by itself, but rather an anti-tamper to be paired with anti-cheats.

High-level View of Byfron​
In this section, I will describe the main features of Byfron. I will go more in depth later in this series. The final goal of this series is to know enough to be able to decrypt Roblox and achieve injection.

A key thing I want to mention here: Many developers in this community think that the current Byfron version is somewhat final and weak; really the current version of Byfron isn't anything special.

What you must remember is that in it's current state, Roblox's main goal is to fix core compatibility issues and ensure it works across different computers. Only after that, can they begin to expand to their full potential. So if you are one of them, prepare for huge surprises in the future.

The Basics​
The first thing most reverse engineers would do is to simply try to disassemble Roblox. Well when you do that, the first thing you will notice is the entry point of Roblox is just an int 3; a debug trap. If you look at the imports, you will notice that there is only one import and its the 'run' export of RobloxPlayerBeta.dll, this is where Byfron resides. You will also quickly notice that the entire .text section of Roblox (where the executable code resides) is just junk. So what's happening here? Well, let's first start with the basics. As you may have assumed, Byfron runs before Roblox's entry point. How is this possible?

You need to think about how the Windows loader works. Before attempting to run any code in the executable, it must first resolve all imported DLLs, load them into the process, and call their entry point (if they have one).

When an imported DLL has an entry point and is called by the Windows loader, it's done in the main thread, meaning it runs before the entry point of the process itself, and the process entry point cannot run until all DLL entry points are finished.

Byfron abuses this Windows feature to run code before Roblox's entry point, making the Roblox executable useless if there isn't something to handle the exception.This is done so Byfron can do critical initialization before anything else, for example they have an internal table storing the original imports of Roblox; before they give control to the original entry point, they will manually load all DLLs that Roblox is using imports from. They also protect some of these imports, but that's a topic for a future part.Now that we've covered the essentials, let's look at some of their "dynamic" features.

As I stated earlier, our main goal is to achieve injection. To do that, you need to allocate executable memory for your module and also create a thread to begin execution. To do this, you will need a full-access process handle open to Roblox, however, Roblox is actually checking for open handles to its process, especially from unsigned programs.

If we naively try to open a handle, it will not stay open for long, this means that external cheats will have to continuously re-open new handles, or use an existing handle opened by something else.If you try to remotely create a thread in Roblox, you'll quickly notice that the thread won't start at all.As I said, detailed reversals about those features will come in future parts.

Finally, if you try to remotely allocate executable memory, you may notice that after about 5 seconds, the protection will change. You should also know that Byfron is constantly scanning all running processes on your computer, if they detect public reverse engineering tools like Cheat Engine, ReClass, IDA, etc, they will make Roblox crash.

In a future part, we will write a tool to hide them, so we can analyze Byfron easier. They also have various detections for hypervisors; that will be covered too.

Outline​
Let's reiterate what we now know. Byfron encrypts the game's code, hides the imports, redirects the entry point, monitors running processes in real-time for known reverse engineering tools, tracks the working set of the process, monitors the threads, checks for the presence of a "malicious" hypervisor, and more, which is beyond the scope of this part.

Final Thoughts
This should give you a basic idea of how Byfron operates at high level.

I didn't cover absolutely everything here, only the key features.
In the next part, I will go into detail on analyzing Byfron, the obfuscations they use in their module, the code encryption over Roblox, how to decrypt it, and finally, how to write a dumper.
‎ ‎ Roblox-Verified-Badge.png me when pls donate​
53358856627_1ff5e2c753_k.jpg
[\MEDIA=vimeo]887631960[/MEDIA]
 Like ReplyReport
Like Reactions:zuazuda, AltScript, szze and 9 others
uni
uni
Active Member
JoinedNov 19, 2023
Messages95
Reputation0
DiscordVerified
Multi Accounts0
Nov 22, 2023
Thread Author
Add bookmark
#2
Dr McDingle • 07-28-2023, 04:42 PM
Analyzing Byfron - Part 2: Core Components​
Introduction​
Previously, we discussed what Byfron is and how it differs from traditional anti-cheats. In this part, we'll analyze their module, learn about the obfuscation techniques they use, examine their initialization routine, and discover how they block the creation of external threads.

Credits​
This thread was co-authored by the following user(s):
gogo1000

Table of contents​
Analyzing Byfron - Part 2: Core Components
Introduction
Credits
Table of Contents
Analysis
Main Obfuscations
Other Obfuscations
Initialization Routine
Instrumentation Callback
LdrInitializeThunk
Overview
Analysis​
Main Obfuscations​
In this section, we'll start our initial analysis of Byfron's binary. We'll look at the obfuscation techniques they utilize, and how to circumvent them when reversing.

First, we'll open the binary in IDA, initially you'll see that all of the executable code is in a .byfron section; as you may have already noticed, Byfron is not packed, it also doesn't employ any sort of virtualization. So what do they do? Let's start out by just clicking on a couple of functions. No matter which function you've chosen, as long as it has control flow, you'll see that IDA will likely fail to disassemble it:

part2_1.png

Byfron implements a few variations of this across all of their functions. This is the most common form of obfuscation that you will see throughout your analysis, and it's easily recognizable.

The seemingly "fake" instruction sequences are also not arbitrary, but follow specific sequences that repeat throughout the binary. Along with this are actual fake instructions, as in dead code, which is present throughout the binary.

Some functions have a jump to a register, where the register is a decrypted address being the instruction proceeding the jump. This is done to break control flow analysis by creating an unconditional jump to an "unknown" location (the instruction after the jump as shown below):

part2_2.png

In the above image, the "jmp rcx" jumps to the proceeding instruction. Above it, rax is a jump table, rcx is the hard-coded offset for the next instruction's relative address, and 4 is the size of each entry in the table.

Some functions unnecessarily store excessive amounts of opaque values on the stack, this is present even in the smaller functions and makes their stack frame massive:

part2_3.png

Other Obfuscations​
Now that we've covered some of the main obfuscation techniques that Byfron uses, let's look at some others.

First, you should know that rather than calling exported functions, Byfron is mainly prone to invoking inlined syscalls, this means many of the exported functions you may have been trying to debug won't be called because they're explicitly invoking syscalls rather than directly calling them.

Now, that isn't to say that they don't call any imports at all, but those that they do call are encrypted then decrypted at runtime, which you will see examples of ahead. Similarly to these encrypted imports, many of the important pointers that Byfron stores are also encrypted then decrypted at runtime, that'll become evident once you start analyzing the hook routines placed on thread initialization.

Let's go back to the inlined syscalls, for now, you can ignore the location of the syscall shown here, this is what it looks like:

part2_4.png

As you can see in the image above, rbx holds the address of their stub, I'll cover how they calculate the stub address in the following section. Here's what that stub looks like:

part2_5.png

After you follow the call, you can see that the syscall invoker stub is written to an RX allocation, or should I say: a view? Let's check out how the syscall invoker stub is acquired, remember how I said it's a view? Let's go back to the loop that's located right before the "call rbx."

part2_6.png

The first thing you should notice here is 2 writes that they perform on [rsi + rcx].

As you can see, rsi holds the base address, and rcx holds the offset. You might be wondering, "what is rsi? "Well, upon looking at it in a debugger, you can see that rsi is actually the pointer to the syscall invoker stub itself... That's weird, it looks like it's writing over the RX allocation for the syscall invoker, but that allocation didn't have any write permission, so how do they do it?

If you look closely, you can see that while the code appears to be the same, they aren't actually the same address, that's what I mean by view. They have 2 separate allocations that are both mapped to the same physical address, but have 2 separate permissions. The allocation currently being written to is RW, and the one used in the calls is RX. Here, we can see a structure containing both of these views:

part2_7.png

So, going back to the syscall invocation. Before they perform the call to the stub, they move the RX view into rbx, it looks like this: mov rbx, [r12 + 08].

It should also be known that they only use one stub, and prior to calls they update the syscall ID, that's what the loop does. Here's the writes that occurred during their LdrInitializeThunk hook, which I'll discuss later:

part2_8.png

That mainly covers the dynamic obfuscation techniques that are used. There are more, but they don't deserve their own separate discussion, they'll be covered in reversals for the next parts.

Initialization Routine​
In this section I'll be covering the first routines that Byfron invokes, these are called prior to Roblox's entry point.

I should note that there are a lot of things that happen during initialization, I won't be covering all of them, just the important parts.

As stated in part 1, one of the first things that Byfron does is load the modules that Roblox imports functions from, and while that might be pretty obvious considering that Roblox only has one import (Byfron's dll), something else happens in this stage.

If you look at the memory regions of the imported modules, you might notice that some of them aren't mapped normally.

part2_9.png

The picture above shows the memory regions for where user32.dll is mapped. As you can see, the allocation type is mostly private, and part of the executable code is mapped.

Normally, loaded modules aren't copied into the private working set of the process, instead they're shared until a process writes into them, which invokes CoW (Copy on Write), but that isn't the topic of this thread.

Basically, this is one of the modules that Byfron "protects," the idea is that you can't patch its code, and if you try to, you will get an error while trying to change its protection to writable.

"Why is this, and how is it possible" is a question that you may be asking yourself.Well, analyzing the function responsible for protecting the module is no easy task, it's massive and has a lot of completely "unrelated" code, which makes manual static analysis nearly impossible.

So, for this I'll be using my hypervisor and exit on each syscall fired by Roblox during the execution of this function.To do this, I set a trap at the beginning to enable my logger, and a trap at the end to stop tracing.

These are the logs captured by my hypervisor during the execution of this function:

Hypervisor trace (char limit)

char + attachment limit ⬇️
Attachments
part2_3.png
part2_3.png
248.1 KB · Views: 330
Last edited: Nov 22, 2023
‎ ‎Roblox-Verified-Badge.png me when pls donate​
53358856627_1ff5e2c753_k.jpg
[\MEDIA=vimeo]887631960[/MEDIA]
 Like ReplyReport
Like Reactions:mge, KingMirai, plusplus and 2 others
uni
uni
Active Member
JoinedNov 19, 2023
Messages95
Reputation0
DiscordVerified
Multi Accounts0
Nov 22, 2023
Thread Author
Add bookmark
#3
part2_extra.png

What you'll see first is Byfron opening a handle to the dll under the knownDlls OB directory.

Then, it queries various bits of information about the dll. After that, it opens a handle to the file itself and creates a section object backed by the file.

Next, you will notice that it creates a view of it with only read permissions and closes the handle to the section object. This is where things get interesting. Then, they create 2 sections with RWX permissions, I should note that the number of sections and the size of them varies per each protected module and these sections are not file backed.

Once the sections are made, it creates views of the sections with RW permissions, and copies the original dll data to them.Interestingly, it then unmaps the real module allocations that the Windows loader created, and maps the same sections again at the exact same addresses that the Windows loader allocated at, but this time without write permissions.

It should be noted that physical address they translate to is the same as the RW sections that were initially created.Now that Byfron has replaced all of the Windows loader allocations with its mapped sections, they allocate memory normally for the rest of the module, this part isn't protected.Finally, the unmap the first file-backed read-only section, as well as the 2 writable sections that they used to write the original module's code. They then close all section handles.

Another important thing to note is that Byfron mapped their views with the MEM_PHYSICAL flag, which prevents you from enabling CoW from usermode. Some of the other modules they protect are: ntdll, kernel32, kernelbase, and other Windows dlls.

Now that we've gotten past the imported module protection, we can discuss what else their initialization routine does.It can now register an instrumentation callback, which from now on I will refer to as "IC," being an acronym.

Wondering what an IC is? It's an undocumented feature within Windows that allows you to capture transitions from kernel mode to user mode. For example, before a syscall returns to the process, Windows checks if there is an IC registered for the process, if so, it sets rip to the address of the callback instead of the intended return address.

ICs allow you to log APCs, exceptions, and thread creations, making them very powerful.Before I move on to the next section, it should be known that Byfron hooks various functions within ntdll, here are some:

part2_10.png

Some of these hooked functions are also redirected in Byfron's IC, meaning if you were to place a breakpoint on one of these functions, it will never trigger because the IC will manually redirect it to their hook directly.

Instrumentation Callback ​
Now that we know what an IC is, where they can be applied, and some elementary knowledge of their use in Byfron, we can now go more in-depth.In this section, I'll talk about the events their IC hooks, and the reason for hooking them.Note that in the current version, their IC ignores all syscall events.

Below is the disassembly of their IC:

part2_11.png

The function in the middle is responsible for setting the return address. Basically, the argument it receives is the original return address and it compares it against various target functions, if the addresses match up, then it returns their hook, otherwise it returns the argument passed.Finally, the IC jumps into the address that was returned.

The functions that are currently targeted are: KiUserExceptionDispatcher, LdrInitializeThunk, and KiUserApcDispatcher.

Also, they set a flag in TEB, they do this to prevent recursion within the thread, if they didn't do this then if their hook triggered an event that is targeted, it'd infinitely recur. They prevent this by not redirecting in their IC if this flag is set (initialization of this flag can be seen below):

part2_12.png

Here's a decompilation of Byfron's IC after fixing the control flow, this is not from the latest version, but everything is the same.

part2_13.png

LdrInitializeThunk​
In the previous section, we uncovered that one of the events Byfron's IC redirects is LdrInitializeThunk.

So, what's so special about this function? Well, when the Windows kernel creates a new thread, it jumps to LdrInitializeThunk, it performs various datum initialization before jumping to the actual thread start address.

This means that Byfron has full control over any new thread as it's created before its start address is ran, and can decide whether or not to let the thread continue, or to terminate it.If you try to create a thread externally, you will realize that it never actually jumps to your thread's start address, this is because Byfron does not let non-whitelisted threads be created.

You might be wondering "what determines if a thread is whitelisted" and that's a good question.Earlier we saw that they hook some ntdll imports, well among those is NtCreateThread(Ex).

This hook exists so that interally created threads by Roblox code or loaded modules will be whitelisted after a few checks, the hook creates a stub that decrypts the start address, then jumps to it. Then, they spoof the start address of the thread to the stub.

By the time the new thread spawns and hits the LdrInitializeThunk hook, Byfron knows if it's legit or not.Here's the disassembly of this stub, you can see it backs up registers, decrypts the real start address, then later it restores them and jumps to the start address:

part2_14.png

Before the stub returns, it pushes rax, which holds the decrypted start address, and when it hits the return it continues to the actual thread entry.

Overview​
In this part we started reversing the Byfron binary, discovered the common obfuscation techniques that are employed, found out how they invoke syscalls, looked at the initialization routine, figured out how their protected modules work, uncovered instrumentation callbacks and how/why Byfron uses them, analyzed the insturmentation callbacks and discovered which events it hooks, and shed light on how Byfron prevents/detects external thread creation.

In the next part we will do a detailed analysis of Byfron's LdrInitializeThunk hook, and figure out how we can successfully create threads externally.

Then, we will figure out how they monitor the working set of the process and also analyze how they encrypt/decrypt Roblox's code in realtime. I'll also explain how they detect various reverse engineering tools and how their hypervisor detections work.

Finally, we'll get started on writing a decryptor and injector and detail the process behind it.
Attachments
part2_12.png
part2_12.png
18.5 KB · Views: 116
Last edited: Nov 22, 2023
‎ ‎Roblox-Verified-Badge.png me when pls donate​
53358856627_1ff5e2c753_k.jpg
[\MEDIA=vimeo]887631960[/MEDIA]
 Like ReplyReport
Like Reactions:KingMirai and plusplus
uni
uni
Active Member
JoinedNov 19, 2023
Messages95
Reputation0
DiscordVerified
Multi Accounts0
Nov 22, 2023
Thread Author
Add bookmark
#4
gogo1000 • 08-01-2023, 11:24 AM

Analyzing Byfron - Part 3: Threads, Memory and Encryption​
Introduction​
In the last part we covered a lot of the core Byfron components, in this part we will continue on analyzing the LdrInitializeThunk hook and cover some other concepts. At the end we will also cover some of the Roblox level protections, that is the ones you will see once you decrypt Roblox and disassemble it.

Previously, we discussed what Byfron is and how it differs from traditional anti-cheats. In this part, we'll analyze their module, learn about the obfuscation techniques they use, examine their initialization routine, and discover how they block the creation of external threads.

Credits​
This thread was co-authored by the following users:
Dr McDingle
AVX

Table of contents​
Analyzing Byfron - Part 3: Threads, Memory and Encryption
Introduction
Credits
Table of Contents
Analysis
LdrInitializeThunk
Memory Scans
Encryption
Stubbed Branches
Import Protection
Overview
Analysis​
LdrInitializeThunk​
In the last part we analyzed how they block external threads, now we will cover how their hook actually works.

To find the hook address you can either follow their hook jmp inside KiUserExceptionDispatcher or look at the IC, because they still hook the actual function. A few instruction below where the jmp goes you will see a call instruction. This is the function that we are interested about.

Now there are many ways we can do to trace whats happening inside it, lets think about it for a moment. First based on logic we can assume they will access the start address of the thread that triggered the hook, based on that we can identify the main logic of the function quite easily. So what we will do is use a hypervisor to single step the function but instead doing it manually, we will only break on instructions that access the start address of our thread.

part3_1.png

In the picture above what you are seeing is compares where they compare our start address with some functions, like you can see, they first check if the start address is any of the LoadLib variations, and then they compare it against 2 more interesting functions.

Now what we need to know is that Windows could create threads 'externally' over some places and they actually do. So Byfron actually can't block every single thread otherwise bad things will happen. One thing they allow threads from is the Thread Pool API, which could create several threads, and the last 2 logs from the trace above are exactly related to that.

If we go to that location we can see there is cmp instruction comparing some value stored on stack with R13, which stores our thread start address.

part3_2.png

The value they are comparing our start address against is actually just decrypted above the compare and the result is stored at [rbp + 560h], which holds the address of TppWorkerThread. This function is unexported so you don't see the real name inside the trace we showed above.

In short, if the thread start address is TppWorkerThread, Byfron actually allows the thread to continue instead terminating it. They have 1 more whitelisted function as you can see on the trace, but it doesn't matter which one we are going to use.So now the question is how we will use that to make our threads, well this function is actually within protected module so you can't easily patch it, same goes for the Byfron module. I have detailed how Byfron protects some modules in the previous part so it's up to the reader to use this knowledge.

Memory Scans​
In the last part we also mentioned that Byfron monitor all Roblox allocations. In short, they whitelist all their internal allocations, they also hook NtProtectVirtualMemory as well as NtAllocateVirtualMemory so they can whitelist memory allocated by internal modules.

Note they only really whitelist/care about executable allocations. Periodically Byfron scans for all the memory in the Roblox process and if they find executable memory that isn't within their map of whitelisted pages they will remove it's permission and wait in hopes of thread execution there, which will raise exception and Byfron will find your thread.

There are many ways to attack this, you could try to hide your allocations but that isn't easy considering their inline syscalls. You could also reverse the hooks I mentioned above and follow the actions they take when you attempt to allocate executable memory.

By any means those functions are massive and pretty hard to follow, but one thing you can notice is they all access a map and adds the allocation there if it passes a few checks. Looking at the NtProtectVirtualMemory hook, I noticed one of the function that it calls seemingly adds a value passed to it inside a map.

After breakpointing it, I noticed that the arg is actually the base of the executable page, the next arg is the size and the third arg is seemingly always 1. The function RVA for the current version is: RobloxPlayerBeta.dll + 0x20207D0 and the version of Roblox I'm currently analyzing is version-b9021bd8128442aa

part3_3.png

As you can see, this map contains list off every single whitelisted executable page inside the Roblox process. Once they do their scans, if the allocation they find isn't there, they will target it.

I tried to make executable alloc and add it to the list, and as expected Byfron didn't change its protection after the scans ran.

Encryption​
In this section, we will delve into the encryption used by Byfron specifically on Roblox code. As mentioned in part 1, the entire .text section of Roblox is encrypted and only decrypted when necessary. Byfron accomplishes this by setting the entire encrypted region as NO_ACCESS. Upon execution, this triggers a no access exception, which then invokes their exception routine.

From there, they check the exception type and exception address to determine the appropriate action to take. If the exception occurs within Roblox code, they perform a few checks and initiate the decryption procedure. However, Byfron has strategically placed traps in certain pages within Roblox that are not actual code but designed to act as traps.

If the execution starts in one of these protected pages, it results in a ban. This tactic prevents someone from decrypting all pages successively. To avoid race conditions, Roblox employs two separate views of memory. The first view, the executable one, is read-only and contains the encrypted code. The second view is writable, allowing them to write and decrypt the pages without altering the initial decryption and effectively avoiding race conditions.

Moving on, let's analyze their exception hook using the same method applied to the LdrInitializeThunk hook. They must access the exception RIP, so we can log all instructions that access it. This will reveal various compares with hardcoded values, as shown in the picture below:

part3_4.png
Last edited: Nov 22, 2023
‎ ‎Roblox-Verified-Badge.png me when pls donate​
53358856627_1ff5e2c753_k.jpg
[\MEDIA=vimeo]887631960[/MEDIA]
 Like ReplyReport
Like Reactions:KingMirai and plusplus
uni
uni
Active Member
JoinedNov 19, 2023
Messages95
Reputation0
DiscordVerified
Multi Accounts0
Nov 22, 2023
Thread Author
Add bookmark
#5
In the image above, you can see that r12 holds the page offset inside the Roblox module, and it is compared against some values. These values are actually the page offsets of the trap pages within Roblox ((Address - Base) / PAGE_SIZE). When decrypting, it is crucial to avoid triggering execution on those specific locations.

Now that we understand this, let's consider how we will begin decryption. The most convenient approach is to do it dynamically. Byfron decrypts the page when execution occurs. They also decrypt pages if someone attempts to read from them, but they restrict using the same reading location on too many pages, making this approach less ideal.

One possible solution is to spawn a thread on every encrypted page inside Roblox, which will force Byfron to decrypt it. However, this presents a new issue. When the thread continues execution after Byfron decrypts the code, it will hit unknown code, leading to a crash.

In the past, Byfron used NtContinue to continue from the decryption, resuming the thread with the provided context. However, to prevent hooking of NtContinue, they switched to using iret and set the context themselves.

The iret instruction remains in the initial dispatcher, making it vulnerable and easy to patch. Similar to the LdrInitializeThunk hook, you can replace it with a jump and control the context used to continue the execution of the thread.

To avoid any crashes, you can simply set the RIP to a return instruction, which will direct the execution back to the CALL instruction that originally executed the code. The code is executed by Roblox multiple times, used for each decryption and exception. To avoid returning to actual code and risking a crash, one can set RCX to a magic value. Any register can be used for this purpose, but RCX is a convenient choice as it can easily be passed as an argument to the CreateRemoteThread function.

Here's an example of how this can be achieved:

C++:
if (Context->Rcx == MAGIC_KEY)
{
// This simulates a return instruction and sets the RIP to
// the return address, which is located on the stack for us.
// As mentioned above, we can simply pass a MAGIC_KEY as the
// argument to CreateRemoteThread, which will be in RCX for us.

Context->Rip = *(uintptr_t*)Context->Rsp; Context->Rsp +=8;

}

With this approach, we should have sufficient understanding of the encryption process. In the next part, we will begin writing a decryptor using the methods discussed above. Subsequent sections will delve into some of the 'Roblox level' protections, such as the stubbed branches scattered throughout the code and the imports.

Stubbed Branches​
If you have ever disassembled Roblox, you have noticed the INT3 breakpoints scattered throughout its binary code.

part3_5.png

Those are always located where the first branching instruction was. This essentially means the first branch of every function is replaced by the INT3's.

Before analyzing how this works, lets first delve into how we stumbled upon this discovery. While we were making our binary rebuilder, we can across the INT3's.

Initially, we thought they are replaced call instructions, but to confirm that, we found the same function in Roblox Studio and determined that they, in fact, constituted branching instructions.

With this newfound knowledge, we set a breakpoint on their exception hook and logged every access to the RFlags member in the CONTEXT structure. This approach was motivated by the realization that, in order to effectively emulate these instructions, Byfron would need to access the flags to figure the appropriate branch to follow.

part3_6.png

Each variation of the emulation handlers was associated with a corresponding entry in a jump table, which is used to invoke the desired handler.

part3_7.png

During my observance of accesses to the CONTEXT structure, I detected an instruction responsible for writing to RIP after the emulation process had finished.

part3_8.png

After tracking back, I noticed that Byfron was reading from an massive array in the .data section. This array contains structure used for describing what each int3 stub replaces, it contains the RVA of the stub, the type of the branch, and even the real instructions that were replaced. This is done so Byfron can restore them on functions that are called often.

part3_9.png

In the image above, the first 4 bytes are the type, the next byte is the size of the real instruction replaced, followed by the actual instruction, and at the end, the RVA to the specific stub is stored.

So in short, Byfron emulates the real branch instruction every time the INT3's are hit, if you want to restore them with the real instructions, you can iterate the array and get the RVA of each entry, and write the original instruction on that place, which as I said is also stored in the array

Import Protection​
Just like before, Byfron also protects the binary through their dynamically protected imports.To kickstart the process, I set a breakpoint on a commonly used import and waited until Roblox triggered it.

Upon further examination, I noticed that the caller was dereferencing the key in a readonly section (rdata) and jumping to it. To debug the code, I simply set a conditional breakpoint on their hook and traced if the ExceptionCode was an access violation. Jumping to the key is an invalid address and triggers a page fault.

Once the import exception handler was hit, Byfron first checks if the exception being ACCESS_VIOLATION is caused by a protected import. If so, it continues through the handler logic. The handler will first decrypt the 'key', which will then become an index inside a specific handler. As mentioned earlier, handlers have a hardcoded list for each possible module that Roblox imports something from.

part3_10.png

Each handler has a hash lookup table containing the hash for a specific import. When the correct handler is identified, the RVA to the hash list in Roblox is stored in the structure. The hash algo used is Fnv1a-32. After the import key is decrypted, the code scans through each handler and checks the imports it handles.

For example, if the ntdll handler handles keys from 0 to 20 and your key is 10, then the ntdll handler will be triggered, and 10 will be used as an index in the hash lookup table. Similarly, if the kernelbase handler handles keys from 50 to 60 and your key is 55, then lookup index 5 in the handler-specific lookup table will be used. After that, they now have the Fnv1a hash, and they will call the GetExport function with it, which will return the specific export once it's found.

part3_11.png

If the module isn't loaded (Roblox uses imports from some modules that aren't loaded when the process loads), then Byfron will manually load it and try using both flags on LoadLibraryExA.

After locating the import, Hyperion then recursively follows every JMP/CALL to locate the lowest point in the chain. Once this process is completed, they set the RIP to the respective import and return back to their exception dispatcher.

Overview​
In this part we figured out a way to make external threads, analyzed how Byfron scans for executable memory, figured out how they encrypt the Roblox code and a way to decrypt it and looked at some of the Roblox level protections.

In the last 3 parts we have covered all the basics and figured out how to successfully make threads, decrypt Roblox and prevent our allocations from being found. In the next part we will analyze how Byfron detects malicious processes and also how they detect hypervisors. And finally we will use all this knowledge to write a example decryptor and injector.
Last edited: Nov 22, 2023
‎ ‎Roblox-Verified-Badge.png me when pls donate​
53358856627_1ff5e2c753_k.jpg
[\MEDIA=vimeo]887631960[/MEDIA]
 Like ReplyReport
Like Reactions:KingMirai and plusplus
uni
uni
Active Member
JoinedNov 19, 2023
Messages95
Reputation0
DiscordVerified
Multi Accounts0
Nov 22, 2023
Thread Author
Add bookmark
#6
gogo1000 • 09-04-2023, 05:32 PM

Analyzing Byfron - Part 4: Hypervisor and process scans​
Introduction​
In the previous 3 parts we have covered nearly all important aspects of Byfron, now we are gonna look at some of the things they do to make reversing more diffucult. That is the Hypervisor detections and reversing tools.

Table of contents​
Analyzing Byfron - Part 4: Hypervisor and process scans
Introduction
Credits
Table of Contents
Hypervisors
EIP overflow
Trap flag
#UD
Process scans
Overview
Hypervisors​
EIP overflow​
Using hypervisor for analysis can be very powerful if applied correctly, it gives you full control of your system allowing you to intercept anything. Because of that Byfron uses various method to detect hypervisors. However, unlike kernel level anticheats, Byfron have limited things that they can do from usermode. Even so, they seems to be sufficient in stopping most people.In general if you have ever written hypervisor from scratch and have followed the SDM correctly then you have nothing to worry about. Sadly most open source hypervisors out there have many emulation mistakes which makes them easily detectable.

Anyone who have written a hypervisor in the past will most likely find their methods within less than a hour. Byfron uses methods to detect specific hypervisors as well as generic ways. To start analysing them, I simply log various information for each vmexit Roblox caused. Now you must remember that I have disabled various vmcs controls to limit the exits I get so i can focus on the basics first.

After printing the exit IP as well as some generic information like the cpu mode and the flags the first thing that caught my eye is an exit from 0xFFFFFFFE with reason CPUID.

{
"Process": "RobloxPlayerBeta.exe",
"Message": "Exit from: FFFFFFFE
| Reason: CPUID
| RAX: FFFD0033
| RCX: FFFD0000
| Long mode:
| Trap flag: 0"
}

At first you might be wondering what is so special about that. Well if you look at the cpu mode, you can see its not in long mode anymore. Now you probably know that 32 bit programs are executed in compatibility mode on 64 bit system.

Byfron takes advantage of this by putting the cpu in 32 bit mode and executing cpuid there, which causes unconditional vmexit. The cpuid instruction is 2 bytes long so in normal operation the EIP will be incremented by 2, but this will cause it to overflow.

Now many open source hypervisors like to assume that they are always running in long mode and use 64 bit instruction pointer internally. Which means the behaviour of incrementing by 2 will be different if a hypervisor like that is loaded than on bare metal.

In any case, exception will be caused, for example in normal operation, the cpu will try fetching next instruction at address 0 and get a page fault, then the OS will invoke the exception handlers with code ACCESS_VIOLATION.

Interestingly enough, Byfron doesn't use their IC to handle this, instead they register a VEH. In you are wondering how I figured this out, I simply followed the strategy used in previous parts. That is to simply use my hypervisor to log accesses on RIP and current code segment within the exception record. After this I got few logs within the same function and upon looking at it in IDA, I immediately knew whats going on.

After fixing the control flow and decompiling it, this is what I saw:

part4_1.png

They are simply checking if the current CS is 32 bit one, if the exception address is 0 and if the reason is ACCCESS_VIOLATION. After this, they simply handle the exception and continue. Now you might ask yourself continue where?

part4_2.png

This is the function that switches to compatibility mode and jumps to 0xFFFFFFFE.Basically if the exception happened correctly, they simply set RIP to next instruction after the call, which in this case jumps to the end of the function, which returns normally.

Now fixing this is very simple, just check the current cpu mode and increment the instruction pointer appropriately.

Trap flag​
Now the other generic check is nothing special or unique. In one of the exits, I noticed the trap flag being set. Now if you have followed the manual correctly, you will know that you are responsible for injecting debug exceptions if the trap flag is set. There are a bunch of side cases with that as well but after looking at the function that caused the exit:

part4_3.png

You can clearly see that they are setting the trap flag, setting ss which changes interruptability state and finally causing unconditional vmexit.

If you don't handle those correctly, the behaviour will change from the one on bare metal.

Heres what the manual says about blocking by ss: "Execution of a MOV to SS or a POP to SS blocks interrupts for one instruction after its execution.” In addition, certain debug exceptions are inhibited between a MOV to SS or a POP to SS and a subsequent instruction.

So in your vmexit handler, you need to reset the state as well as inject the debug exception, which in this case will be skipped but if you don't reset the interruptability state, 2 instructions will be skipped.
#UD​
Another thing I don't think is worth getting into detail, but is worth mentioning, is that they will also cause various #UD exceptions.

Now some hypervisors disable syscall/ret extension which makes them cause #UD. They then capture the exception in the hypervisor to log syscalls. However some of them assume that all #UDs coming from usermode are syscalls and emulate it. But in Byfron's case, the cause of the #UD is not real syscall. To fix this you should simply read the instruction and verify its a syscall, if not simply inject #UD back to guest.
Process scans​
Now as mentioned before, they also scan running processes for specific reversing tools. Some example of those are: "Cheat engine, Reclass.NET, IDA" and more.

This makes reversing and testing things much more difficult specially if you rely on those tools. Luckily enough its pretty easy to go around the scans. Byfron have dedicated thread that runs every second, scanning for those tools.After finding the thread and isolating syscalls from it, you can see the sequence they are taking. First they query '\\Device\' directory by using NtOpenDirectoryObject and NtQueryDirectoryObject to scan for specific devices, then they also query the "BaseNamedObjects" directory of the current session. This one have interesting reason for. You might not know but IDA actually makes a few named objects.

part4_4.png

The reason for those scans is exactly that, they search for them and if they find them, they will crash the client.

The next syscalls invoked are quering "\\REGISTRY\\MACHINE\" KeyValueBasicInformation. Getting SystemPoolTagInformation and SystemBigPoolInformation from NtQuerySystemInformation, as well as SystemExtendedProcessInformation. After that they start firing NtUserFindWindowEx to find every window on your system.

After they find all windows, they get the owning process and try to open handle to it with PROCESS_QUERY_LIMITED_INFORMATION, if the handle is successfully opened, they use NtQueryInformationProcess with ProcessImageInformation flag and close it.

Then they wait for a second and repeat the steps. Its important to note they also do a different scans every once in a while, one of which gets all running threads as well.

They also scan for open handles, so you should avoid having a FULL_ACCESS handle opened to Roblox, especially from unsigned process.

In the end, if you don't want to bother much, you can simply hook NtOpenProcess from kernel and deny access to whatever program you are trying to hide. For IDA, you must also hook NtQueryDirectoryObject and check if they are quering the IDA object, if so, skip it.
Overview​
In this part we learnt how Byfron detects hypervisors and scans for specific processes. Now as I have covered the most important concepts of Byfron, in the next part, I will discuss approaches in decrypting and dumping Roblox, and in the end, write a working decryptor.




0AVX0 • 09-05-2023, 12:25 PM

For anyone wondering, a fix for the EIP enforcement may look like:​
C++:
uintptr_t OldRip = Hv::VmRead(VMCS_GUEST_RIP);
uintptr_t NewRip = OldRip + Hv::VmRead(VMCS_VMEXIT_INSTRUCTION_LENGTH);

if (!Hv::ReadCs().LongMode)
        NewRip &= 0xFFFFFFFF;

Hv::VmWrite(VMCS_GUEST_RIP, NewRip);

You could apply further optimizations by implementing code to check if NewRip exceeds the maximum possible EIP value first, and then act accordingly if so.

This works because when the CPU isn't in long mode, it isn't using RIP, and thus can't store that value.Keep in mind, you shouldn't be injecting the debug exception normally, and should instead leverage the pending debug exceptions field in vmcs.
