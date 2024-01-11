# 编译器原理

## Introduction of Compiler

### Overview of Compiler

#### What it compiler?
Compiler is a program that can tranlate source code to another code, and **save it as executable**.
Source code: C/C++, Java, C#, HTML...
Object code: x86, ARM, MIPS...

#### What's the difference between compiler and interpreter?
An interpreter is program that reads a program and produces the results of executing that program. And will **not save the compiled executable**.

### The Structure of Compiler
Most of the compiler contains three ends: **Front End**, **Middle End**, **Back End**.
![Alt text](Images/IMG_0752.PNG)
The basic workflow of a compiler:
* Front end maps legal source code into IR(intermidiate representation)
* Middle end optimizes the IR
* Back end maps IR into target machine code
* Admits multiple passes(read from the begin of the code to its end)

1. Front End
    The structure of Front End is like this
    ![Alt text](Images/IMG_0754.PNG)
    The responsibility of scanner is map character/byte stream into tokens. What is token? Token is something like `<id, x>`, for example, I have a cpp code looks like
    ```cpp
    x = x + 2;
    ```
    In scanner stage, it will be translate into
    ```token
    <id, x> <op, => <id, x> <op, +> <number, 2>
    ```
    Now we have a problem, how do we specify the **legal** tokens? The first step is to recognize if the identifier is legal or not
    * Regular expression, only words derived from regex are acceptable
    ![Alt text](Images/IMG_0755.PNG)
    Then we need to judge the whole expression is legal or not
    * Deterministic Finite Automaton(DFA)
    * Longest match principle
    ![Alt text](Images/IMG_0756.PNG)
    
    Then we need to send those tokens to **Parser** to generate a parse tree, to do that, we can use **Contex-Free Grammer(CFG)**. In Compiler, we often use **Abstract Syntax Tree(AST)**. In AST, we will put identifier and number in leaf node, and operation in non-leaf node. Just like this
    ![Alt text](Images/IMG_0757.PNG)
    Also, during the process of builing AST, the compiler will also check if the usage of number and identifier is legal or not.

2. Middle End
    Middle End stage is an optimizer, the main responsibilities of it are:
    * Analyzes IR and rewrites(ro transforms) IR
    * Primary goal: refuce running time of the compiled code
    * Other goals: improve space, energy consumption
    * Must preserve the **meaning** of the code

    But what kind of optimization will middle end use? One of it is subscript, for example, assuming that we have a C code below
    * Subscript
        ```c
        for(int j = 0; i < n; j++) {
            A[i][j] += C;
        }
        ```
        In memory, 2 dimensional array is stored just like 1 dimensional array, we need to calculate the offset of the element so that we can access the element.
        So actually we will have this kind of calculation in each loop:
        ```c
        for(int j = 0; i < n; j++) {
            addr(A(i, j)) = addr(A(0, 0)) + i * (row size) + j
            A[i][j] += C;
        }
        ```
        But in this expression, some calculation will not be changed during this loop
        ```c
        addr(A(0, 0)) + i * (row size) // this part will always be the same throughout the whole loop
        ```
        So, we can 


3. Back End
    The overview of Back End stage:
    ![Alt text](Images/IMG_0758.PNG)
    The responsibilites of Back End stage:
    * Translate IR into target machine code
    * Choose instructions to implement each IR operation
    * Find a good order for the instructions to execute
    * Decide which value to keep in register