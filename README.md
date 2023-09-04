# Solidity-memory-safe
### Explaining Solidity memory safe annotation


   The `memory-safe` assembly annotation is used to indicate to the Solidity compiler that an assembly block respects Solidity's model of handling and using memory and that it is safe for the Solidity compiler to carry out optimizations.

   An assembly block is considered `memory-safe` if it only accesses memory that was previously allocated by Solidity, expands memory and updates the free pointer accordingly, or accesses the scratch space.

   However, using the `memory-safe` annotation is mostly relevant when compiling smart contracts via the [Intermediate Representation code generation path (new codegen or IR-based-codegen)](https://docs.soliditylang.org/en/v0.8.21/ir-breaking-changes.html). The `--via-ir` code generation path can perform memory optimization and can also prevent stack too deep errors by moving stack items to memory as stated in the [Solidity docs](https://docs.soliditylang.org/en/v0.8.21/assembly.html#memory-safety) (although this does not happen all the time).

   Memory optimizations and moving stack items to memory to avoid stack too deep error with `--via-ir` are disabled by default for assembly blocks. This is because the compiler is not sure if Solidity's memory handling model is followed or not. But we can indicate that we follow this model by adding the `memory-safe` annotation, like this:

   ```solidity
   assembly ("memory-safe") {
       //...
   }
   ```

   If this annotation is added to an assembly block, the compiler can add optimizations to an assembly block when necessary.



  As stated earlier, the `memory-safe` annotation is mostly relevant when compiling smart contracts via the Intermediate Representation (IR) code generation path. The annotation indicates to the compiler that it can carry out memory optimization for an assembly block (when compiling via IR), but it does not affect the assembly code result directly. No matter the optimization, the compiler will not change the implemented logic inside an assembly block, but may change how it is carried out. This means that if memory is misused inside an assembly block, with or without the `memory-safe` annotation, the effect of the misuse will hold. Misusing memory in this case can include writing to memory and not updating the free memory pointer, or writing to memory offsets that already contain data, leading to loss of data.

  Here is an example usage of the `memory-safe` annotation:



  ```solidity
    function pocWithMemory() external pure returns (string memory) {
            string memory data;
            assembly ("memory-safe") {
                mstore(0x80, 0x0000000000000000000000000000000000000000000000000000000000000013)
                mstore(0xa0, 0x537472696e677320696e20617373656d626c7900000000000000000000000000)
                mstore(0x40, 0xc0) // update free memory pointer
                data := 0x80 // store the length of the string. this will make `data` return the values we store in memory which is a string ("Strings in assembly").
            }
            return data;
        }
  ```


Here, we adhere to Solidity's convention of using memory by writing after the current free memory pointer `(0x80)` and updating it at `mstore(0x40, 0xc0)`. However, removing the `memory-safe` annotation here does not affect the output of the assembly code. It may affect the optimization of the compiler (only when compiling with `--via-ir`), but not the output.

  It is not easy to have a scenario or case where `--via-ir` compilation optimizes to move stack items to memory.

  Here, we will incorrectly use memory, add the `memory-safe` annotation, and compile via IR for memory optimization, but we will still get incorrect data. This is true whether or not we add the `memory-safe` annotation here, compile via IR, or compile with normal solc. This is because the assembly block is not implemented accurately.

  ```solidity
    function pocWithMemoryMisuse() external pure returns (string memory) {
            string memory data;
            assembly ("memory-safe") {
                mstore(0x80, 0x0000000000000000000000000000000000000000000000000000000000000013)
                mstore(0xa0, 0x537472696e677320696e20617373656d626c7900000000000000000000000000)
                // mstore(0x40, 0xc0) // we are meant to update free memory pointer here
                data := 0x80 // store the length of the string. this will make `data` return the values we store in memory which is a string ("Strings in assembly").
            }

            string memory data_2 = "Overrides"; // this will override data because we didn't update the free memory pointer.
            return data;
        }
  ```

  Thus, we can say that the `memory-safe` annotation does not guarantee that the assembly code is actually `memory-safe`. The compiler can only optimize the assembly code if it is confident that the code does not misuse memory. We add the `memory-safe` annotation to indicate to the compiler that we believe the code does not misuse memory.
