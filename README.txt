Report

As part of the human brain project BrainScaleS is a unique project on many levels. This includes a processor solely used by the HICANN DLS, which manages synaptic weights for every neuron built into one of the many wafers. To accelerate the speed at which this so called plasticity processor unit (PPU) computes all synaptic weights of every neuron used, the processor has an extended instruction set architecture (ISA) that supports vector registers and single input multiple data (SIMD).
This report deals with the task of adding built-in functions to an existing backend of GCC, specifically the one used by the PPU, in order to extend the already implemented set of functions according to the users needs

Contents
- Compiler-structure
    - in regard to the PPU
    - Basic tasks of a back-end
- Insn-Coding and Register Transfer Language
- Adding a built-in type
- Argument handling
- Special cases
- Namespace conventions
- Conclusion


Compiler Structure

Most Compilers that are used nowadays are built of three basic components which handle different steps in the process of converting human-readable programming language to machine-readable machine code. As does the GNU Compiler collection (GCC) which also can be seen as made of three main parts. The so called front-end, middle-end and back-end.
All three parts work more or less independently from each other and communicate over a compiler-specific „language“, which is described a the Intermediate Representation (IR). It is typically never seen by the user and exists for a fact in many different forms [reference] one of which is Register Transfer Language (RTL), which is the lowest-level IR used by GCC. It is the most interesting IR when working on a back-end and will get more attention later in this report. But in before that we need to understand the structure an functioning of a compiler in general.

The first mentioned Front-end resembles the main interaction point between the human programmer and the machine. Front-ends are usually divided by their respective programming languages such as C, C++, Java…  and have the main task of converting any programming language into unified IR, that can be passed to the middle-end in GIMPLE or GENERIC language. Therefore no matter which language you prefer, in an ideal case code, that is syntactically identical, should not differ after it is processed by the front-end. This is due to the goal of compilers such as GCC and LLVM to support as many languages and machines as reasonably possible while offering the equally good optimization and saving themselves overall work. It obvious that a single compiler for every combination of language and machine would simply not be practical, especially as the optimization taking place in the middle-end follows the same rules for pretty much any architecture.
The middle-end main task is basically this sort of optimization, and makes for the main difference between compilers as most compilers offer the same range of front-ends and back-ends. (Part about the optimizations taking place in the middle-end). After all optimizations are through, the middle-end passes IR in form of RTL to the back-end. As you can see the middle-end rarely needs to be modified except for fundamental changes in the compilers architecture such as new kinds of optimizations and „multiple memory handling“ (also Harvard-Architecture (vielleicht))
The back-end is responsible for the final steps of the compilation process as it translates the general RTL IR into specific Assembly commands. It uses some sort of table of available assembly instructions, that is provided, and finds the best fitting instructions. GCC for example uses a Lisp-like language (is this RTL?) that uses something called insns. These combine different properties with the  Assembly commands like an equivalent set of actions that are executed, the  operands and the constraints it must satisfy. These will be further explained later. The back-end also contains the the code which implements processor-specific built-in functions.
Depending on the Compiler architecture the back-end it finally emits IR to the assembler which emits the machine code in assembly or emits assembly itself. Then finally the linker links the assembly code of all program files together and substitutes the offset addresses with absolut addresses to generate the final machine code.
reload
register spilling
register handling
endianess
wordsize


For the rest of this report we will deal more specifically with the combination of the C front-end and the RS/6000 back-end which is also used for the Power ISA which itself is the basis for the PPU. As exemplary built-in functions we will use the AltiVec vector extension for the rs/6000 architecture because the PPU has its own vector extension as well and ultimately this is meant to help extend the set of vector functions for the PPU.

Insn-Coding and Register Transfer Language

When implementing a new builtin function, it is usually necessary to start on the lowest, most abstract level there is in a back-end which is the machine description. It contains the definiton, expansion and splitting of insns. First it is important to understnd what an insn is.
An insn is a single expression of which the RTL is composed. The insns themselfs are doubly linked and build a chain of instructions which is the RTL. It is usually easier to describe an insn equivalent to an instruction but insns can also be used as jump-labels for declaration of the code or dispatch tables for switch statements. but in our case we are most interested in using an insn as an instruction. In this case next all insn have a pattern which describes the side-effects of the insn/assembly command it is associated with. Here are two simple examples of such patterns:
- load pattern
- add pattern
first you will probably have noticed the name of the insn which is mostly quite descriptive but optional if you do not want to use it as an builtin-function.
As you can see next the name ist followed by a pattern which fist of all represents the effects and side effects of an insn. e.g. how many registers are used? which register will be changed whch remain the same etc. But it contains more information than just the side effects of an insn. the pattern also handles operands and constraints:
Operands are not only the input or output of an insn they can also describe temporarily used registers or constants. in most cases though they either represent an input or output of an Assembly macro. Each operand is „matched“ by match_operand and must fulfill certain conditions.
(match_operand:m n predicate constraint)
m desribes a mode the operand must match such as SI (Single Integer), SF(Single Float) or vector modes such as V8HI (vector consisting of 8 Half Integer Elements). these modes can be bundled into groups such as all vector integer modes (VI) but must match each other for known instructions such as „add“.
Next n is the operand number. It s chosen by the developer and stays the same for multiple uses of the same operand and helps identifiying the assmebly operand with the insn operand.
The predicate is a string that further defines conditions an operand must fulfill such as being a constant, being in a certain range or being a certain vector type. It may also check if the operand is of a certain register type. all predicates are defined in predicates.md, where existing predicates can be altered or new ones can be added.
The constraint finally does not help matching an operand but already gives certian rules for the register class the operand is going to have. Most people get to know constrains when working with Assembly code. The letters that identify with a certain register class such as r for general register, f for floating point register and m for memory operand are the same contraints that are used when defining insn.

The next line of defining an insn is a more or less simple boolean statement that decides whether the insn is available or not. The easiest case is having a TARGET variable such as TARGET_ALTIVEC here. This means that if the AltiVec extension is activated → TARGET_ALTIVEC = true → the insn is available.

The next line is calles the output template or output statement and states the assembly macro associated with the insn. you can see a „%“-sign followed byy the opernad index n for every argument the assembly macros accepts. The number and type of opernads for each assembly macro is stated in the opcode section of binutils. This can also be a piece of C code to decide between different assembly macros.

At last the insn attribute completes the insn. it is optional but helps the compiler notice key effects a insn has, such as being a load r vector load instructon, an arithmetic or logical instruction etc. there is a wide variety of options here but only a few are needed in our case.

All this information makes up just a single insn in a very long chain of equally complex insn that make up the RTL. But all of this information is needed for the compiler to first find the perfect insn to do certain tasks and second optimize the RTL code as good as possible. This why it is usually worth putting some effort into the RTL-like patterns of insns. it is importnat to notice that the assmebly macro is executing what was defined by the chip developer either way. The pattern is just meant to represent the effects of the assembly macro as good as possible.

When the RTL is finally generated it mostly resembles those patterns of the insn defintion as you can see in the following example the loads two vectors from memory into vector regsiters, add the vectors and stores the result into the memory as a third vector.

Because the RTL is recurring at different points when working with a back-end, here are a few basic examples of RTL and basic RTL „commands“



We now want to add our own insn which will later be used as a built-in function of the compiler. We choose to implement an imaginary built-in function that multiplies a vector v with a scalar number s.
…
We now finished our very own insn which will later become a fully supported built-in function.


Adding a built-in type

To allow the compiler to differ which insns we acutally want to use as a built-in. The rs/6000 back-end has a special file which contians macros for every builtin function that will be available. These  macros will be loadad in the rs6000.c file and also handled there. But before this can happen we have to create a built-in macro for our newly created insn.
WE must first differ between the different kinds of macros. the rs/6000 back-end already provides us with templates for differnt altivec built-ins. These templates basically help identifying the right macros later on and avoid naming conflicts.
Next we have to differ between the number of input opernad our built-in function is going to have and the number  in th etemplate name is meant to be equal to th enumber of input operands . all Macros expect output operand except for the X-names macros. which are special cases we will emphasize later.
We now choose BU_ALTIVEC_2 because our newest insn has to input operands and add the following lines:

the first argument is the name of the macro and will be used by the compiler internally. the user only gets interact with the second string which is the new built-ins name for the front end. (for altivec built-ins the tempalte adds __builtin_altivec_ in front of the given name). Next the attribute for the builtn must be given. typically this CONST but for builtins that load or store in the memory this is MEM for example. The last argument is the name of the insn we decided on earlier.

You can also add an „overload“ macro for your builtin as this allows for type checking and reducing builtins for different vector modes to just on final builtin.
here we first need a Macro name and second the function name we want to use later (this functon name is preceeded by __builtin_vec_)


Argument handling

After all builtin macros and overloaded macros are initialized, we want to assign the argument types of our builtin functions. This is quite similar to the argument type in normal functions but looks quite differently. We actually need to mention every argument type that our builtin supports. This means we need to differ between signed and unsigned variables as well as different vector and integer sizes.
Hence we must not forget any valid combination of output and input operands that could come up.
To achive this the file rs6000-c.c build a connecton between our overloaded macros and the builtin macros we created earlier. Htis contains an array of altivec_builtin_types called altivec_overloaded_builtins. the altivec_builtin_types struct consits of the macro codes from rs6000-builtins.def and 4 signed chars that are the return typ and up to three input types. since we only need to the third will always be zero. 
For every argument type available there is a macro defined that is named also named after the argument type. e.g. RS6000_BTI_V16QI for a signed 16 element quarter inetger vector, RS6000_BTI_unsigned_INTSI for an unsigned singel integer, ~RS6000_BTI_INTQI for a pointer adress of a signed quarter integer.

Now we need to go through every combination of input and return types there is for our builtin.
…

Special Cases

not all cases can be handles by the aforementioned method. for example a builtin function wihout a return type. in  htis case we use the BU ALTIVEC_X template thaat slightly differs from other tmeplates……

Nameing conventions

After we fully implemented our builtin functions we will finalize the functions by defining their final names. There are two naming conventions we can follow and typically it is advised to follow both in order to offer the user a set of alternatives. The first naming convention is the nameing convention already used by the AltiVec extension which starts every builtin function with vec_ and chooses a general descriptive name which is derived from the used Assembly macro if possible.
This is the most intresting case for our exemplary Altivec builtin function. because there is already a vec_mul builltin we will choose the descriptive abbreviation vec_smul for scalar multiplication.

The other naming convention gets intresting when extending the backend with another vector extension. If there were an S2PP vector extension, all assembly macros would begin with fxv which is a good alternative for the vec_ prefix altivec uses for its builtins. then it is a good idea so offer all assembly macros with their already known names e.g. fxvaddbm would become fxv_addbm. for more genral builtins that combine the halfword and byte macro it is recommended to leave out the letter h or b likewise. rendering fxv_addm the builtin function that works with both V8HI and V16QI vectors.

Conclusion


This report emphasizes on the process of including the extended ISA in the GNU Complier Collection (GCC) to simplify the usage of the processor for future purposes and providing documentation for any user interested in contributing to this task.

Contents
*introduction
**current implementation
**why GCC?
**parts of the compiler
**preparation
*implementing s2pp into GCC
**front-end
**middle-end
**test routine
*conclusion
*outlook

-sync() is a problem?

current implementation
In the beginning of this internship the PPU had had only the most basic implementation possible. This allowed only for the use of assembly instructions via the "asm_volatile" function. For practical purposes the functions were substituted by macros that allowed a more simplistic use of operations though one still needed to address specific registers and store/load data manually to/from the these registers. This made programs using these macros quite complex and hard to "read" although some users tried using macros to increase readability to a certain point.
Still this bears another problem which is the maintenance of such programs. E.g. Adding an additional value somewhere in the middle of the program might require substantial rearrangement of registers for complex programs. Also it is very likely that programs written in this way are ineffective because they undergo no optimization at all.
This brings up enough motivation to further implement the s2pp processor into GCC and eliminate many of these disadvantages.


Why GCC?
Currently GCC marks the most common compiler available and supports most CPU architectures available to this day. This includes the PowerPC architecture the PPU is based on. Naturally one would therefore extend the implementation of the PowerPC architecture with the added instructions rather than writing the whole package from scratch.
If done correctly, a complete GCC implementation has some advantages over simply defining macros for each instruction.
The most significant is the advantage in computing time achieved by allowing GCC to optimize the code rather than forcing human written instruction into the assembler code that cannot be optimized. Especially for the intended use of the HICANN DLS speed is relevant factor as well as usability.
Because GCC is so widespread, it is likely that every user of the system is familiar with GCC to some extent which makes the first steps of programming on this platform far easier.
Also the documentation for GCC is quite good and continuously updated as GCC itself.
Though it is not expected that the plugin provided at the end of this internship will be updated as frequently and supports mainly the GCC version used in the group network.
The main let down of GCC is the shier complexity of GCC itself. It can be quite hard the get used to the way GCC is written which includes the register translate language (RTL)

Parts of the compiler
A complier is usually described as three parts: the front-end, middle-end and back-end.
The main purpose of the compiler is translating code, written in a programming language such as C or C++, into machine code, which the processor can actually read.

Front-end

Middle-end

Back-end

Preparation

The very beginning of this internship involved getting "comfortable" using the PPU and learn the very basic concepts behind its construction. For this reason there were a couple days used to write simple programs that used the s2pp instructions just to get to know the way they worked at that point. It was obvious that two things, the register allocation and the manual store and load operations were both obsolete work a user had to do while programming.
Before there was a chance to modify the compiler though, there was the need for a new instance of the cross-compiler to not interfere with other people compiling "stuff" in the network.

Building the experimental s2pp-cross-compiler

This was the first step towards 