# Dead code elimination (DCE) and Aggressive dead code elimination (ADCE) passes in LLVM

LLVM version 8.0.0.

## Code Locations

/include/llvm/Transforms/Scalar/DCE.h

/lib/Transforms/Scalar/DCE.cpp

/include/llvm/Transforms/Scalar/ADCE.h

/lib/Transforms/Scalar/ADCE.cpp

/lib/Transforms/Utils/Local.cpp

## Dead code elimination (DCE)
Dead code elimination (DCE) is a transformation in LLVM which deletes the instructions which have no impact on the result of the program. These instructions are either not reachable, or the results of the statements are not usable or these instructions have no side effects. 

The main idea behind DCE is to mark an instruction as live if the variable defined in the instruction has any use. Otherwise, it is marked as dead. The dead instructions are then removed from the program. LLVM uses a worklist-based algorithm for DCE, which builds on top of Dead Instruction Elimination (DIE). DIE makes single pass over the function, removing instructions that are dead. 

DCE uses a SmallSetVector for the implementation of the worklist and the algorithm will take linear time in the number of instructions in the program. 


### Example

**C Code:**
```c
int foo(int x, int y)
{
  int a = x + y;
  a = 1;
  return a;
}
```

**The SSA form looks like:**
```ll
define i32 @foo(i32 %x, i32 %y) #0 
{
  %1 = add nsw i32 %x, %y
  ret i32 1
}
```
**After running DCE on the above SSA form:**

```ll
define i32 @foo(i32 %x, i32 %y) #0 
{
  ret i32 1
}
```

### Limitations

DCE, however, misses out on some cases, where even though the instructions are perceived as live, they are not contributing to the result of the program. This limitation is resolved by the following transformation pass called Aggressive Dead Code Elimination (ADCE).


## Aggressive Dead code elimination (DCE) 

The Aggressive dead code elimination (ADCE) pass in LLVM takes a different approach to removing dead instructions. It assumes all instructions are dead unless proved otherwise. So, rather than identifying dead instructions, it identifies live instructions. The remaining instructions are assumed to be dead and removed from the program. Similar to DCE, it uses a worklist-based algorithm. 

Algorithm: 

1.	Iterates over the instructions and adds the instructions which are live to the worklist. An instruction is considered live if: 
    1.	It is the output statement (e.g., return)
    2.	It has known side effects (e.g., assignment to global variable or calling a function with side effects)
    3.	It defines a variable x used by an already live statement
    4.	It is a conditional branch, and some other, already live statement is control dependent on the branch (and its block)
2.	Pop values from the worklist, identify the defining instructions for the operands. 
3.	Mark these instructions as live and add to the worklist.  
4.	Repeat until worklist is empty.
5.	Remove all unmarked statements.

ADCE uses the following data structures: MapVector, DenseMap, SmallVector, SmallSetVector, SmallPtrSet, std::pair. Also, the algorithm will take linear time in the number of instructions in the program. 

### Example

**C Code:**

```c
int foo()
{
  int a = 0;
  for (int i=0;i<4;i++)
  {
    a = a + i;
  }
  return 0;
}
```

**The SSA form looks like:**

```ll
define i32 @foo() #0 
{
  br label %1

; <label>:1                                      
  %i.0 = phi i32 [ 0, %0 ], [ %6, %5 ]
  %a.0 = phi i32 [ 0, %0 ], [ %4, %5 ]
  %2 = icmp slt i32 %i.0, 4
  br i1 %2, label %3, label %7

; <label>:3                                       
  %4 = add nsw i32 %a.0, %i.0
  br label %5

; <label>:5                                      
  %6 = add nsw i32 %i.0, 1
  br label %1

; <label>:7                                      
  ret i32 0
}
```

On running DCE, it does not delete the phi node for ‘a’ as it is marked as live subsequently. Whereas on running ADCE, since ‘a’ is not marked as live based on the conditions above, the phi node for ‘a’ is removed after running ADCE. 

**The result of running ADCE followed by loop deletion is:**

```ll
define i32 @foo() #0 {
  br label %1

; <label>:1                                     
  ret i32 0
}
```
