---
title: MIPS Calling Convention for CS64
---


## Outline

<ol>
<li><a href="#preface">Preface</a></li>
<li><a href="#motivation">Motivation</a></li>
<li><a href="#key_control_flow">Key Instructions and Transferring Control Flow</a></li>
<li><a href="#arguments">Arguments</a></li>
<li><a href="#return_values">Return Values</a></li>
<li><a href="#function_additional_registers">Functions that Need Additional Registers</a></li>
<li><a href="#functions_call_functions">Functions that Call Functions</a></li>
<li><a href="#functions_call_functions_additional_registers">Functions that Call Functions Which Need Additional Registers</a></li>
<li><a href="#mapping_variables_to_registers">Mapping Variables to Registers</a></li>
<li><a href="#invariants">Invariants to Preserve</a></li>
</ol>
    
<h2><a id="preface">Preface</a></h2>
<p>
      This document discusses the <i>calling convention</i> which we will be using in CS64, which defines the protocol used for calling functions and returning from functions.
      Many different such calling conventions exist, and this document merely describes one of them.
      For those interested, this document's calling convention is based off of the calling convention <a href="https://courses.cs.washington.edu/courses/cse410/09sp/examples/MIPSCallingConventionsSummary.pdf">here</a>.
</p>
<p>
      Relative to other calling conventions, the calling convention in this document is simplified.
      Specifically, this calling convention:
</p>
<ul>
<li>Does not utilize the <code>$fp</code> and <code>$gp</code> registers, which other conventions use to manage things like global variables and stack-allocated arrays</li>
<li>
        Assumes that functions take no more than four arguments.
        Other calling conventions handle this by putting additional arguments on the stack before calls.
        This inherently makes things more complex, especially in circumstances when different functions which push additional arguments on the stack may be called.
        For the purposes of this class, this restriction is ok, as we will never have a case of a function taking more than four arguments.
</li>
<li>
        Assumes that functions return no more than two values.
        Similar to the problem with arguments, other calling conventions will handle this by putting additional values on the stack when they return.
        This makes things more complex, particularly when different functions are in play which return different things.
</li>
<li>
        Assumes that individual values allocated on the stack are always 32 bits in length.
        This means that we will never deal with stack-allocated arrays or C <code>struct</code>s.
        Such allocations can be handled without changing any of the rules described in this document, but they have intentionally been omitted to keep things simple and streamlined.
</ul>
<p>
      In addition, relative to some of the calling conventions out there, this one is somewhat optimized, particularly when it comes to memory usage.
      This is intentionally done to reduce boilerplate code that always needs to be written, and to help keep code focused on the task at hand as opposed to bookkeeping activities.
      This calling convention is optimized in the following ways:
</p>
<ul>
<li>
        When it comes to preserved registers, we explicitly preserve only those registers we use.
        This not only reduces the amount of code necessary, it overall simplifies code, and it makes function calls more memory-efficient.
</li>
<li>
        We do not allocate extra space in order to ensure that stack allocations are always in uniform-looking blocks.
        With respect to how this calling convention works, there is no requirement that stack allocations be of certain sizes, and so we don't enforce anything here.
        This saves on memory, and tends to make programs easier to reason about, as less memory is in play.
</li>
</ul>
<p>
      The rest of this document describes why we need a calling convention, and how the particular calling comvention used in this class works.
      In all cases, when you are asked to implement something that obeys the MIPS calling convention, this means implementing something in accordance to the rules specified in this document.
<b>In general, solutions that violate the rules specified in this document will receive no credit.</b>
      The reason why this is so severe is because solutions which violate the calling convention tend to be completely untestable; if we call into your code as part of a test harness, and then your code violates the calling convention, it means that the state of the test harness has likely been overwritten, meaning it will no longer work correctly.
      While we have a way of detecting if this happened, we do not have a way to correct things once the damage has been done.
      It is for this reason that it is so important that your code behaves according to the calling convention specified in this document.
</p>
    
<h2><a id="motivation">Motivation</a></h2>
<p>
      The goal of any calling convention is to have a way of implementing functions in assembly.
      This is more challenging than it may initially sound, as there is nothing inherently built into the processor that allows for function calls to be performed, at least in a way that is as straightforward as adding numbers together.
      For example, with MIPS, while we have a way to start executing some other piece of code (as we need for a function), we do not have an inherent way of jumping <i>back</i> to some other piece of code when execution has completed elsewhere.
      As another example, while we have a way to save variables somewhere with MIPS (i.e., with registers), we don't have an inherent way to save an <i>arbitrary</i> number of variables.
      Without this capability, we can have neither arbitrarily nested function calls, nor can we have recursion.
</p>
<p>
      While MIPS does not have built-in capabilities which handle functions, it does provide enough functionality to implement functions ourselves.
      For example, MIPS allows for memory access, which, when used with care, can be used to implement a call stack.
      Additionally, with some care, we can explicitly have code jump back to a caller, effectively implementing a <code>return</code> in the process.
      As such, we are not stuck in the water when in comes to functions; it's just that we need to put more effort into their implementation, as functions exist at a higher level of abstraction than what is provided by the MIPS hardware.
      This also means that there are many possible ways to implement functions, each with its own set of advantages and disadvantages.
      For correctness, it is important that all functions are implemented in the same way, as function calls require specific interactions between different pieces of code.
</p>
<p>
      For reasons which will soon become clear, registers are very special in the calling convention.
      This is ultimately why the different registers have the names they do (e.g., <code>$t0</code> versus <code>$s0</code>).
      As such, for starters, restrict your knowledge of registers to only <code>$zero</code>; we will introduce the other registers, along with their intended usage, piecewise.
</p>

<h2><a id="key_control_flow">Key Instructions and Transferring Control Flow</a></h2>
<p>
      This section introduces register <code>$ra</code>.
</p>
    
<p>
      There are two instructions which are key to performing function calls:
</p>
<ul>
<li>
<b><code>jal</code></b>: A specialized jump instruction.
        Jumps to a particular label, much like <code>j</code> does.
        What is special about <code>jal</code> is that it will also put the address of the <i>next</i> instruction into register <code>$ra</code>.
</li>
<li>
<b><code>jr</code></b>: Another specialized jump instruction.
        This will jump to the address contained in the specified register.
</li>
</ul>
<p>
      The above two instructions work in tandem with each other.
      For example, consider the following function:
</p>
<pre>
void emptyFunction() {}
</pre>
<p>
      In order to call <code>emptyFunction</code>, this would be done like so:
</p>
<pre>
jal emptyFunction
</pre>
<p>
      The above code snippet will jump to the code contained after the <code>emptyFunction</code> label (which we have yet to show), and put the address of the <i>next</i> instruction into register <code>$ra</code>.
      The actual definition of <code>emptyFunction</code> is below:
</p>
<pre>
emptyFunction:
  jr $ra
</pre>
<p>
      Since <code>emptyFunction</code> does not do anything, it simply immediately returns using the <code>jr</code> instruction.
      Recall that <code>jal</code> put the address of the next instruction to execute after the function returns into register <code>$ra</code> (short for &ldquo;return address&rdquo;).
      As such, <code>jr $ra</code> ends up jumping to the address of the instruction immediately after the call.
      This effectively achieves the transfer of control flow between a call and return - we jump to the definition of the function on a call, and we return to the point immediately after the function call using the address that was specified when the call was made.
</p>

<h2><a id="arguments">Arguments</a></h2>
<p>
      This section introduces registers <code>$a0 - $a3</code>.
</p>
<p>
      To make our definition of <code>emptyFunction</code> more useful, we will have it take arguments.
      We change the definition to the one below:
</p>
<pre>
void takesArguments(int x, int y) {
  x = x + y;
}
</pre>
<p>
      While this definition does not actually do much of anything yet, it suffices as a minimal example to illustrate how a call is made.
      Consider the following function call to <code>takesArguments</code>:
</p>
<pre>
takesArguments(3, 4)
</pre>
<p>
      The above function call can be implemented like so:
</p>
<pre>
li $a0, 3
li $a1, 4
jal takesArguments
</pre>
<p>
      That is, arguments are specified in registers <code>$a0</code> through <code>$a3</code>.
      The first argument is passed in <code>$a0</code>, the second argument in <code>$a1</code>, and so on.
      The way values are passed is to actually set the corresponding register to the appropriate value.
      In this case, because the first argument was a <code>3</code>, we set register <code>$a0</code> to the value <code>3</code> <i>immediately before</i> the call to <code>takesArguments</code>.
      As for the definition of <code>takesArguments</code>, it will then look at <code>$a0</code> and <code>$a1</code> to get its parameter values.
      With this in mind, <code>takesArguments</code> has the following definition below:
</p>
<pre>
takesArguments:
  add $a0, $a0, $a1
  jr $ra
</pre>
<p>
      The only difference from our previous <code>emptyFunction</code> definition here is the addition of the <code>add</code> instruction, which effectively computes <code>x = x + y</code>.
      The variable <code>x</code> is passed in register <code>$a0</code>, as it is the first argument.
      Similarly, the variable <code>y</code> is passed in register <code>$a1</code>, as it is the second argument.
</p>
<p>
      Keep in mind that we only have registers <code>$a0 - $a3</code> available to us, meaning functions can take at most four arguments (which would be passed in all four registers).
      You should never need more than four arguments in this class.
      There is a way to pass more via other mechanisms, though we won't be discussing them in this class.
</p>
    
<h2><a id="return_values">Return Values</a></h2>
<p>
      This section introduces registers <code>$v0</code> and <code>$v1</code>.
</p>
<p>
      As it stands, <code>takesArguments</code> does not have much of a practical purpose, as it merely adds its parameters together.
      What would be far more useful is to return something.
      As such, we modify the definition of <code>takesArguments</code> to yield the definition of <code>addTwo</code> shown below:
</p>
<pre>
int addTwo(int x, int y) {
  return x + y;
}
</pre>
<p>
      For function returns, registers <code>$v0</code> and <code>$v1</code> are used.
      The first value returned is placed in register <code>$v0</code> (if applicable), and the second value returned is placed in register <code>$v1</code> (if applicable).
      With this in mind, we can implement <code>addTwo</code> in MIPS assembly, like so:
</p>
<pre>
addTwo:
  add $v0, $a0, $a1
  jr $ra
</pre>
<p>
      The returned value, that of adding <code>addTwo</code>'s two parameters together, is put into register <code>$v0</code>.
      Calling code then expects that <code>$v0</code> will hold the returned value.
      This is very much like issuing the system call to SPIM for reading an integer from a user (<a href="https://www.doc.ic.ac.uk/lab/secondyear/spim/node8.html"><code>syscall</code> 5</a>), where the integer read in is placed in register <code>$v0</code>.
</p>

<h2><a id="function_additional_registers">Functions that Need Additional Registers</a></h2>
<p>
      This section introduces <code>$t0 - $t9</code>.
</p>
<p>
      So far, the functions we have implemented have been relatively simple.
      Let's make things a bit more complex, by introducing the need for temporary variables, like so:
</p>
<pre>
int needsTemps(int x, int y) {
  int add = x + y;
  int sub = x - y;
  int mult = add * sub;
  return mult;
}
</pre>
<p>
      The function <code>needsTemps</code> above uses the temporary variables <code>add</code>, <code>sub</code>, and <code>mult</code>.
      Previously in this sort of situation, you simply would use some arbitrary registers to use here.
      However, for the purposes of following the MIPS calling convention, for reasons we'll get into shortly, we are a bit restricted with what we can use.
      In this case, we can use registers <code>$t0 - $t9</code>.
      For translating the above code, we'll use the following temporary variable to register mapping.
      Other than the fact that we restrict ourselves to registers <code>$t0 - $t9</code>, this mapping is arbitrary; we can use any of the variables in the range <code>$t0 - $t9</code>.
</p>
<table border="1">
<tr>
<th>Original Variable</th>
<th>Mapped to Register</th>
</tr>
<tr>
<td><code>add</code></td>
<td><code>$t0</code></td>
</tr>
<tr>
<td><code>sub</code></td>
<td><code>$t1</code></td>
</tr>
<tr>
<td><code>mult</code></td>
<td><code>$t2</code></td>
</tr>
</table>
<p>
      With the above mapping in hand, we can translate <code>needsTemps</code> into the following MIPS assembly code:
</p>
<pre>
needsTemps:
  add $t0, $a0, $a1
  sub $t1, $a0, $a1
  mult $t0, $t1
  mflo $t2
  move $v0, $t2
  jr $ra
</pre>
<p>
      The above MIPS code is a straightforward translation of the C code, which follows the C code exactly.
      The only portion introduced from the previous example is that now we can use registers <code>$t0 - $t9</code> if we need additional registers.
</p>
    
<h2><a id="functions_call_functions">Functions that Call Functions</a></h2>
<p>
      This section introduces register <code>$sp</code>.
</p>
<p>
      Here we introduce the portion of the MIPS calling convention which is arguably the most crucial, but also the most difficult to understand: functions which call functions.
      To illustrate why things suddenly become more complex here, consider the following C code:
</p>
<pre>
void third() {}
void second() {
  third();
}
void first() {
  second();
  exit();
}
</pre>
<p>
      Using the components you've seen so far, you may be tempted to implement <code>first</code> and <code>second</code> using the following MIPS assembly code:
</p>
<pre>
third:
  jr $ra

second:
  jal third
  jr $ra

first:
  jal second

  # perform the exit syscall from SPIM
  li $v0, 10
  syscall
</pre>
<p>
      The above code, however, does <b>not</b> work correctly.
      To understand why, recall that <code>$ra</code> is intended to hold the address of the instruction to return back to.
      When <code>first</code> calls <code>second</code>, the address of <code>first</code>'s instruction after the call (specifically the <code>li $v0, 10</code> instruction) is put into register <code>$ra</code>.
      This becomes problematic when <code>second</code> calls <code>third</code>, because <code>second</code> then puts the next's instruction into <code>$ra</code> (specifically the <code>jr $ra</code> instruction in <code>second</code>).
      This overwrite's <code>$ra</code>'s value.
      When <code>third</code> returns, this is not a problem, but now <code>second</code> no longer has the appropriate address to return to, as the appropriate address was overwritten when <code>second</code> called <code>third</code>.
      Now, when <code>second</code> executes <code>jr $ra</code>, it's going to start executing instructions where <code>$ra</code> currently points, which just so happens to be the <code>jr $ra</code> instruction in <code>second</code>.
      As such, this code will now forever jump to the <i>same</i> instruction over and over again - the <code>jr $ra</code> instruction in <code>second</code>.
</p>

<h3>A (Broken) Workaround</h3>
<p>
      A potential workaround is to put the address to return to in another register.
      To illustrate this idea, consider the code below, which shows changes from the previous code in <b>bold</b>.
      Note that this is just to illustrate the idea; you should <b>never</b> implement your code this way, as it violates the calling convention.
</p>
<pre>
third:
  jr $ra

second:
<b>move $t0, $ra</b>
  jal third
  jr <b>$t0</b>

first:
  jal second

  # perform the exit syscall from SPIM
  li $v0, 10
  syscall
</pre>
<p>
      The key change with the code above is that <code>second</code> copies the value of <code>$ra</code> into <code>$t0</code> <i>before</i> the call to <code>third</code>, and then it returns to <code>first</code> by jumping to the address specified in <code>$t0</code>.
      This ends up working around the original problem, because now <code>$ra</code> isn't being overwritten.
</p>
<p>
      The problem with this workaround is that it does not scale up to larger programs.
      For example, let's add another function call into the mix, which is named <code>fourth</code> below.
      We will perform the same workaround as before.
      (Again, do <b>not</b> implement your own code this way; this is strictly for illustration purposes only.)
</p>
<pre>
fourth:
  jr $ra

third:
  move $t1, $ra
  jal fourth
  jr $t1

second:
  move $t0, $ra
  jal third
  jr $t0

first:
  jal second

  # perform the exit syscall from SPIM
  li $v0, 10
  syscall
</pre>
<p>
      With the above code, we now use registers <code>$t0</code> and <code>$t1</code> as a sort of spillover for register <code>$ra</code>, in <code>second</code> and <code>third</code>, respectively.
      Crucially, we had to add in a second register into the mix, namely <code>$t1</code>.
      An additional call (say, to <code>fifth</code>) would require an additional register (e.g., <code>$t2</code>) and so on.
      Eventually, we'd completely run out of registers, as MIPS only has 32 registers in all.
      Moreover, this strategy doesn't allow for recursive functions, which need to jump to the <i>same</i> function label over and over again.
</p>

<h3>Using a Call Stack</h3>
<p>
      The solution to the above problem, a solution implemented widely, is that of a stack in memory.
      You have likely heard of a <a href="https://en.wikipedia.org/wiki/Call_stack">call stack</a> before; this is ultimately where the call stack comes in.
      The call stack is a sort of data structure in memory, which stores the return addresses of the various functions which have been called in the past.
      When a function is called, the address that the function needs to return to is stored (or more specifically, pushed) onto the call stack.
      When a function returns, the address is taken from (or more specifically, popped off of) the call stack.
      Psuedocode illustrating this process with our previous example is shown below (note that <code>push</code> and <code>pop</code> are not real MIPS instructions; this is just for illustration purposes).
</p>
<pre>
fourth:
  jr $ra

third:
  push $ra
  jal fourth
  pop $ra
  jr $ra

second:
  push $ra
  jal third
  pop $ra
  jr $ra

first:
  jal second

  # perform the exit syscall from SPIM
  li $v0, 10
  syscall
</pre>
<p>
      In the above psuedocode, the intended semantics of <code>push $ra</code> is to push the current value of <code>$ra</code> onto the call stack.
      The intended semantics of <code>pop $ra</code> is to pop the topmost value of the call stack into register <code>$ra</code>.
      To visually show how <code>push</code> and <code>pop</code> effectively work, the series of images below demonstrate this.
</p>
<p>
      Initially, the stack is empty, like so:
</p>
<img src="{{'image/empty_stack.jpg' | relative_url}}">
<p>
      When <code>first</code> is initially entered, it calls <code>second</code>.
<code>second</code>'s first action is to push the value of <code>$ra</code> onto the stack, which dictates where <code>second</code> should return to.
      With this <code>push $ra</code> instruction, the stack ends up looking like the following:
</p>
<img src="{{'images/push_1.jpg' | relative_url }}">
<p>
<code>second</code> then calls <code>third</code>, which subsequently pushes the value of <code>$ra</code> onto the stack with <code>push $ra</code>
      Keep in mind that with <code>second</code>'s <code>jal third</code> instruction, the value in the <code>$ra</code> register now holds where <code>third</code> should return to.
      As such, after this second <code>push $ra</code> instruction, the call stack looks like the following:
</p>
<img src="{{ '/images/push_2.jpg' | relative_url }}">
<p>
<code>third</code> then calls <code>fourth</code> with the <code>jal fourth</code> instruction, jumping to the code at the label <code>fourth</code>.
      As <code>fourth</code> doesn't need to call any other functions, <code>fourth</code> doesn't need to push anything onto the call stack.
      Instead, <code>fourth</code> can directly return to <code>third</code> using the value <code>third</code> placed into register <code>$ra</code> with <code>jal fourth</code>.
      This jump is achieved in <code>fourth</code> with <code>jr $ra</code>.
</p>
<p>
      With this use of <code>jr $ra</code> in <code>fourth</code>, execution the proceeds to <code>third</code>, specifically at the instruction <code>pop $ra</code>.
      The <code>pop $ra</code> instruction in <code>third</code> puts the value on top of the stack into register <code>$ra</code>, resulting in a stack that looks like the following:
</p>
<img src="{{ '/images/push_1.jpg' | relative_url }}">
<p>
      The next instruction executed is <code>third</code>'s <code>jr $ra</code> instruction, which uses the value freshly popped off of the call stack into register <code>$ra</code>.
      With this, control is transferred back into <code>second</code> (that is, <code>third</code> returned into <code>second</code>), specifically into <code>second</code>'s <code>pop $ra</code> instruction.
      With this one instruction, the value on top of the call stack is placed into register <code>$ra</code>, resulting in a call stack that looks like the following:
</p>
<img src="{{ '/images/empty_stack.jpg' | relative_url }}">
<p>
      The value in register <code>$ra</code> now holds the address of the instruction that <code>second</code> should return to, specifically <code>first</code>'s <code>li $v0, 10</code> instruction (well, whatever instructions the psuedoinstruction <code>li $v0, 10</code> translate to).
<code>second</code> then executes its <code>jr $ra</code> instruction, transferring control back into <code>first</code>.
      From there, the value <code>10</code> is loaded into register <code>$v0</code>, and ultimately the prorgram exits with SPIM's exit system call via the <code>syscall</code> instruction.
</p>

<h3>Using a Call Stack in Full MIPS Assembly</h3>
<p>
      The previous example demonstrates how to use the call stack to store return addresses as execution proceeds through different functions.
      While this example illustrates the high-level concepts, it does not actually explain how to implement it purely in MIPS assembly, as it relies on the <code>push</code> and <code>pop</code> instructions, which are not valid instructions or even psuedoinstructions.
      MIPS instead offers an even lower-level interface to the call stack, handled via a new register: <code>$sp</code>.
      What <code>$sp</code> holds is a pointer to the top of the stack; <code>$sp</code> is, in fact, short for &ldquo;stack pointer&rdquo;.
      Instead of using our <code>push</code> instruction, which simultaneously allocated space on the stack and stored a value into the newly allocated space, we must instead explicitly move the stack pointer in one step and store the value of interest in another step.
</p>
<p>
      Moving the stack pointer is easily accomplished via the <code>addiu</code> instruction.
      To allocate space for <code>$ra</code>, a four byte (32 bit) value, we'll need to adjust the stack pointer (held in <code>$sp</code>), like so:
</p>
<pre>
addiu $sp, $sp, -4
</pre>
<p>
      We use <code>addiu</code> as opposed to <code>addi</code> because there is no such thing as a negative memory address.
      That is, all memory addresses are unsigned, so triggering a processor-level exception on overflow (as <code>addi</code> does) is nonsensical (as there is no such thing as overflow in an unsigned context).
      As for why we decremented <code>4</code> as opposed to incrementing <code>4</code> is because on MIPS, the stack, by convention, grows from high addresses to low addresses.
      That is, an empty stack would be represented by the stack pointer (<code>$sp</code>) holding a large value like <code>0xFFFFFFFF</code>, and a full stack would be represented by the stack pointer holding a small value like <code>0x00000000</code>.
</p>
<p>
      The above instruction effectively allocates space on the stack.
      The next step is to store the value of <code>$ra</code> into the newly allocated space.
      This can be performed with the <code>sw</code> instruction, like so:
</p>
<pre>
sw $ra, 0($sp)
</pre>
<p>
      The above instruction will copy the value of <code>$ra</code> into the memory address at <code>$sp</code> with offset <code>0</code>, which corresponds to the space we just allocated.
</p>
<p>
      To summarize, our psuedocode instruction:
</p>
<pre>
push $ra
</pre>
<p>
      ...is equivalent to:
</p>
<pre>
addiu $sp, $sp, -4
sw $ra, 0($sp)
</pre>
<p>
      As for the <code>pop</code> psuedocode instruction, this is effectively a <code>push</code> in reverse.
      Instead of allocating space and then storing something at the newly allocated space, we load the value at the top of the stack and deallocate the slot at the top of the stack.
      The loading of the value off of the top of the stack can be performed with the <code>lw</code> instruction, shown below, which loads the value into register <code>$ra</code>:
</p>
<pre>
lw $ra, 0($sp)
</pre>
<p>
      From here, we can deallocate the space at the top of the stack, which is performed by moving the stack pointer <code>4</code> bytes to a higher address (indicating a less deep stack).
      This is shown below:
</p>
<pre>
addiu $sp, $sp, 4
</pre>
<p>
      In summary, the pseudocode instruction:
</p>
<pre>
pop $ra
</pre>
<p>
      ...is equivalent to:
</p>
<pre>
lw $ra, 0($sp)
addiu $sp, $sp, 4
</pre>
<p>
      With the above pieces in place, we return to our original example, subbing in actual valid MIPS instructions as opposed to our <code>push</code> and <code>pop</code> pseudoinstructions:
</p>
<pre>
fourth:
  jr $ra

third:
  addiu $sp, $sp, -4
  sw $ra, 0($sp)
  jal fourth
  lw $ra, 0($sp)
  addiu $sp, $sp, 4
  jr $ra

second:
  addiu $sp, $sp, -4
  sw $ra, 0($sp)
  jal third
  lw $ra, 0($sp)
  addiu $sp, $sp, 4
  jr $ra

first:
  jal second

  # perform the exit syscall from SPIM
  li $v0, 10
  syscall
</pre>
      
<h2><a id="functions_call_functions_additional_registers">Functions that Call Functions Which Need Additional Registers</a></h2>
<p>
      This section introduces registers <code>$s0 - $s7</code>.
</p>
<p>
      The example above shows how to manipulate the call stack in order to have arbitrarily deep chains of function calls working properly.
      However, it does not jive well when it comes to functions which need additional registers for processing.
      For example, consider the following C code:
</p>
<pre>
int subTwo(int a, int b) {
  int sub = a - b;
  return sub;
}
int doSomething(int x, int y) {
  int a = subTwo(x, y);
  int b = subTwo(y, x);
  return a + b;
}
</pre>
<p>
      The explanations above stated that additional registers were needed, then we could use registers <code>$t0 - $t9</code>.
      However, this is problematic with the above code.
      For example, within <code>doSomething</code>, we may want to map variables <code>a</code> and <code>b</code> to registers <code>$t0</code> and <code>$t1</code>, respectively.
      This mapping would mean that <code>subTwo</code> could not map variable <code>sub</code> to either register <code>$t0</code> or <code>$t1</code>, as doing so would interfere with the values that <code>doSomething</code> puts into registers <code>$t0</code> and <code>$t1</code>.
      While we can get around this problem in this case by being sure to map <code>sub</code> to a different register (e.g., <code>$t2</code>), this is another hacky workaround (which, incidentally, you should <b>never</b> do in your code, as it violates the MIPS calling convention).
      With enough function call nesting, we'll once again eventually run out of registers.
</p>
<p>
      Once again, the call stack comes to the rescue here.
      In addition to pushing the values of <code>$ra</code> onto the stack when we want to preserve them across calls, we can <i>also</i> push other values, as with variables <code>a</code> and <code>b</code> above.
      This is shown in the MIPS code below, which implements <code>subTwo</code> and <code>doSomething</code> legally according to the MIPS calling convention:
</p>
<pre>
subTwo:
  # map sub to $t0 
  sub $t0, $a0, $a1
  move $v0, $t0
  jr $ra

doSomething:
  # allocate space for $ra, x, and y, which all must live past
  # the first call to <code>subTwo</code>
  addiu $sp, $sp, -12

  # save these values onto the stack.
  # note that we use different offsets, which is because everything
  # was allocated in a single large block
  sw $ra, 8($sp)
  sw $a0, 4($sp)
  sw $a1, 0($sp)

  # x and y are already in the right place for the first call to
  # subTwo, so we can call it directly
  jal subTwo

  # load x and y into the right positions for the next call
  # to subTwo.  Note that we intentionally flip where we place
  # these values, as the call is to subTwo(y, x)
  lw $a0, 0($sp)
  lw $a1, 4($sp)

  # allocate space on the stack for a, which needs to survive
  # past the second call to subTwo
  addiu $sp, $sp, -4

  # store a on the stack, which is held in $v0
  # (where subTwo(x, y) returned)
  sw $v0, 0($sp)

  # make the second call to subTwo
  jal subTwo

  # the result of subTwo(y, x) (b) is in $v0.
  # add this to the previously stored a, which we'll need to grab
  # from the stack
  lw $t0, 0($sp)
  add $v0, $v0, $t0 # prep return value directly

  # load in the value of $ra from the stack.  Note that the offset
  # is different from before, as we since allocated additional values
  # on the stack
  lw $ra, 12($sp)

  # deallocate space on the stack, and return
  addiu $sp, $sp, 16
  jal $ra
</pre>
<p>
      The above code is valid, and it obeys the calling convention.
      In general, you should remember the following:
</p>
<blockquote>
      Values in unpreserved registers (every register introduced so far except for <code>$sp</code>), are <b>not</b> preserved across calls.
      You must <b>always</b> assume that non-preserved registers will change values across calls, <b>even if you know for a fact that they don't.</b>.
</blockquote>
<p>
      The above convention means that functions operate independently of each other when it comes to how registers are treated.
      Violations of the above rule in correct code necessarily mean that one function relies on another function not modifying a register, which leads to hacky code (functions should be able to change indepdently of each other).
</p>
<p>
      While the above code works and obeys the MIPS calling convention, it is <i>much</i> more complex than it needs to be.
      Nearly all of the instructions in <code>doSomething</code> are devoted to manipulating the call stack, which is not ideal.
      Additionally, reasoning about which values are on the call stack at which times is not trivial.
      For example, while the value of <code>$ra</code> is stored onto the stack initially at offset <code>8</code> of <code>$sp</code>, we later load the value of <code>$ra</code> from offset <code>12</code> of <code>$sp</code>, as a <code>4</code>-byte allocation had taken place since that point.
      It would be much nicer if we could reference individual variables in the code via registers, which are more permanent than offsets from the stack pointer (<code>$sp</code>).
</p>
<p>
      It is here where registers <code>$s0 - $s7</code> come in.
      These registers are <i>preserved</i>, meaning that their values persist across calls.
      This is in stark contrast to registers <code>$t0 - $t9</code>, whose values do not persist across calls.
</p>
<p>
      The downside of registers <code>$s0 - $s7</code> is that if you want to use one of them, it is up to you to save the register's current value and then later restore it.
      This way, calling code never &ldquo;sees&rdquo; the value of an <code>$s*</code> register change.
      In contrast, with registers <code>$t0 - $t9</code> can be written to directly at all times with no ill consequence to the MIPS calling convention, as nothing is permitted to assume that the values in these registers persist across calls.
</p>
<p>
      To illustrate how registers <code>$s0 - $s7</code> are used, the previous example is rewritten to use them:
</p>
<pre>
# note that subTwo's definition remains the
# same, as subTwo does not call anything
subTwo:
  # map sub to $t0 
  sub $t0, $a0, $a1
  move $v0, $t0
  jr $ra

doSomething:
  # map x to $s0
  # map y to $s1
  # map a to $s2
  # optimization: because b doesn't need to live beyond
  # a call, we do not map it to an $s* register

  # allocate space for the $s* registers we will use,
  # along with $ra
  addiu $sp, $sp, -16

  # save the register's values
  sw $s0, 0($sp)
  sw $s1, 4($sp)
  sw $s2, 8($sp)
  sw $ra, 12($sp)

  # initialize $s0 and $s1 with x and y, respectively
  move $s0, $a0
  move $s1, $a1

  # call subTwo with x and y, which are already in place
  jal subTwo

  # move the result to $s2
  move $s2, $v0

  # call subTwo with y and x
  move $a0, $s1
  move $a1, $s0
  jal subTwo
  
  # compute final result
  add $v0, $v0, $s2

  # restore registers, deallocate, and return
  lw $ra, 12($sp)
  lw $s2, 8($sp)
  lw $s1, 4($sp)
  lw $s0, 0($sp)
  addiu $sp, $sp, 16
  jr $ra
</pre>
<p>
      The above code now utilizes registers <code>$s0 - $s7</code> in order to have values effectively survive past the calls to <code>subTwo</code>.
      This has the following advantages over the prior implementation, among others:
</p>
<ul>
<li>
        Code related to stack manipulation is now in distinct blocks, as opposed to being intermixed with business logic.
</li>
<li>
        Offsets are now uniform; <code>$ra</code> is stored at offset <code>12</code>, and subsequently loaded from offset <code>12</code>.
        It is generally much more easy to reason about code which manipulates the stack from uniform offsets.
</li>
<li>
        The code does not repeatedly need to load values from the stack in between calls; they are already in the <code>$s*</code> registers.
        In this simple example this doesn't mean much, though if we had many calls to <code>subTwo</code> then we'd end up performing much, much more stack manipulation.
        The use of <code>$s*</code> registers here leads to better performance, because we only manipulate the stack when we absolutely have to.
</li>
</ul>

<h2><a id="mapping_variables_to_registers">Mapping Variables to Registers</a></h2>
<p>
      In previous sections, the mapping of variables to registers was given.
      This section outlines a strategy for mapping variables to different registers.
</p>
<p>
      If a variable's value does not need to live beyond a function call, then it can be safely mapped to registers <code>$t0 - $t9</code>.
      While there is no restriction on which registers you map to here, it is generally preferable to map variables to registers in the order that they appear in the program.
      For example, with the program:
</p>
<pre>
void foo() {
  int x = 2 + 2;
  int y = 7 * 8;
}
</pre>
<p>
      ...perhaps the most straightforward mapping is to map <code>x</code> to <code>$t0</code> and <code>y</code> to <code>$t1</code>.
</p>
<p>
      If a variable's value <i>does</i> need to live beyond a call, then generally it should be placed in one of the <code>$s*</code> registers.
      Again, it is generally better to map variables to registers in the order in which they appear in the original program.
</p>
    
<h2><a id="invariants">Invariants to Preserve</a></h2>
<p>
      This section summarizes the different invariants you must preserve in the MIPS calling convention.
      This is redundant with the previous section, but more explicit with respect to exactly what you have to do.
      Violation of <b>any</b> of these invariants constitutes a violation of the MIPS calling convention, and can easily lead to a 0 on an assignment, as it makes code effectively untestable.
      In an exam setting, breaking the calling convention in a setting where code must follow the calling convention once is an automatic 50% point reduction; breaking it multiple times in the same setting will further reduce the number of points awarded.
</p>
<p>
      Without further ado, the invariants you must preserve are listed below:
</p>
<ol>
<li>
        The values that are held in preserved registers (<code>$s0 - $s7</code>, and <code>$sp</code>) immediately before a function call must be the same immediately after the function returns.
        While a call is occurring, the values in preserved registers are allowed to change, but the values must return to their original values after a call is performed.
</li>
<li>
        The values that are held in unpreserved registers (<code>$t0 - $t9</code>, <code>$a0 - $a3</code>, and <code>$v0 - $v1</code>) must always be assumed to change after a function call is performed.
        That is, after a function call is made, the programmer must assume that the values in unpreserved registers all changed in a completely unpredictable way.
</li>
</ol>

