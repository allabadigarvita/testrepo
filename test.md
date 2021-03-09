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


### Examples

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

