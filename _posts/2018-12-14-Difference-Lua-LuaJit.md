
***Lua***:
Lua programs are not interpreted directly from the textual Lua file, but are compiled into bytecode, which is then run on the Lua virtual machine. The compilation process is typically invisible to the user and is performed during run-time, but it can be done offline in order to increase loading performance or reduce the memory footprint of the host environment by leaving out the compiler.

The main advantages of precompiling chunks are: faster loading, protecting source code from accidental user changes, and off-line syntax checking.

Precompiling does not imply faster execution because in Lua chunks are always compiled into bytecodes before being executed. luac simply allows those bytecodes to be saved in a file for later execution.

Precompiled chunks are not necessarily smaller than the corresponding source. The main goal in precompiling is faster loading.



***LuaJIT***:

LuaJIT is a Just-In-Time Compiler (JIT) for the Lua programming language. Lua is a powerful, dynamic and light-weight programming language

As with every performant system, the answer in the end comes down to two things: algorithms and engineering. LuaJIT uses advanced compilation techniques, and it also has a very finely engineered implementation. For example, when the fancy compilation techniques can't handle a piece of code, LuaJIT falls back to an very fast interpreter written in x86 assembly.

LuaJIT gets double points on the engineering aspect, because not only is LuaJIT itself well-engineered, but the Lua language itself has a simpler and more coherent design than Python and JavaScript. This makes it (marginally) easier for an implementation to provide consistently good performance.

For LuaJIT 2.0, the whole VM has been rewritten from the ground up and relentlessly optimized for performance. It combines a high-speed interpreter, written in assembler, with a state-of-the-art JIT compiler.

