# Introduction to Hyperion

**Hyperion** or **Byfron**? Byfron is the original brand that created their anti-tamper product called "Hyperion" which is the anti-tamper, but Roblox bought their product and fully acquired it, so you should name it "Hyperion". Hyperion is likely categorized as an anti-cheat, but it is not really. It is described as an anti-tamper and unlike traditional anti-cheats that prioritize detecting and blocking cheating behaviors, Hyperion's primary objective is to prevent tampering with Roblox's executable code and protect its runtime environment from unauthorized modifications.

The main point of Hyperion relies on its anti-tamper architecture, designed to make reverse engineering, debugging, and tampering a challenge for anyone. Initially, we all thought it would just be a common anti-tamper like any other, but we were wrong. They did way more than just being an anti-tamper, and you'll understand it later by reading this post.

## Main Components

There is a lot to say about these components, so we will divide them into multiple parts.

### Obfuscation Techniques

1. **Fake Instruction Sequences**:
   - They fill their executable code with arbitrary sequences that confuse static disassemblers.
   - Their fake instructions often follow specific and repetitive patterns, making it harder for reversers to know which ones are functional code or filled junk. This will cause tools like IDA or BinaryNinja to misinterpret the code, leading to incomplete or inaccurate disassembly results.
   
2. **Dead Code Sequences**:
   - These do not contribute to actual functionality but complicate reverse engineering by inflating their stack frames and obscuring control flow, creating functions that appear more complex than they really are.
   
3. **Unconditional Jumps to Decrypted Addresses**:
   - They complicate control flow by occasionally jumping to dynamically decrypted addresses, breaking the linear flow of code and forcing reversers to resolve or emulate each jump's destination in real time.

### Memory and Import Protection

1. **Dynamic Import Encryption**:
   - Instead of relying on basic static import tables, they dynamically encrypt import addresses and decrypt them only when necessary.
   - This protection is powerful as the imports are protected by both encryption and trap mechanisms that trigger crashes if accessed improperly.

2. **Memory Monitoring and Page Protection**:
   - They actively monitor executable memory pages in Roblox Memory.
   - They use hooks to manage memory allocations and hook on syscalls like `NtProtectVirtualMemory` and `NtAllocateVirtualMemory` to restrict by whitelisting specific executable memory regions.

3. **Mapped Views for Syscall Invocations**:
   - They use dual views of memory for syscall invocations, divided into RW and RX sections, allowing managing memory without revealing its code structure.
   - They resolve imports through a hash lookup system where each key is hashed using the Fnv1a-32 algorithm.

### Initialization Routine

1. **Pre-execution Control via Windows Loader**:
   - Hyperion's code is executed before Roblox's main code, enabling them to set up protections, encrypt the `.text` section, and set up hooks.

2. **Manual Load and Protection of Imports**:
   - They manually load and protect important imports that Roblox relies on, including creating custom memory sections for these libraries.

3. **Instrumentation Callback (IC)**:
   - They register an Instrumentation Callback, an undocumented Windows feature, to monitor and control threads, manage exceptions and prevent unauthorized actions in the code.

### Memory Management & Protection

1. **Dynamic Code Encryption and Controlled Decryption**:
   - Hyperion encrypts the entire `.text` section of Roblox's executable, making the code unreadable until specific points during execution.
   - They use JIT decryption during runtime to protect important functions, requiring the code to be decrypted at runtime only.

2. **No-Access Memory Protections**:
   - These trigger exceptions when unauthorized access attempts are made. Unauthorized memory access triggers exceptions managed by Hyperion’s handler.

3. **Trap Pages and Execution Verification**:
   - Randomly placed trap pages in memory cause crashes if triggered, preventing brute-force-based decryptors.

4. **Periodic Memory Scans**:
   - Hyperion performs periodic memory scans to ensure the integrity of the executable memory pages.

### Advanced Protections

1. **INT3 Breakpoints**:
   - Placed at branching instructions, INT3 breakpoints interrupt the code flow to prevent debugging or tracing.
2. **Nanomite Technique**:
   - INT3 breakpoints replace the first branch of most functions. Exception handlers conditionally restore the original instruction if execution times indicate normal behavior.

### Hypervisor & Virtualization Detections

1. **Hypervisor Presence Check**:
   - By forcing the CPU into compatibility mode and executing specific instructions, Hyperion detects hypervisors using EIP overflow.
2. **Trap Flags and #UD Exception**:
   - Trap flags set in specific registers cause unconditional VMExits, detecting hypervisors that mishandle this state. Similarly, #UD exceptions detect improper syscall/ret instruction emulation.

### Scanning Processes

Hyperion periodically scans for debugging and reverse engineering tools, monitoring process names and handles. It also inspects named objects and registry keys.

### Limitations

1. **Hyperion’s User-Mode Constraints**:
   - Being user-mode only, Hyperion cannot detect rootkits, kernel manipulations, or hardware-based hypervisors.

### Common Ways to Bypass Their Checks

1. **Abusing Instrumentation Callback**:
   - Unmap Hyperion, hook needed syscalls, and remap Hyperion.
2. **Manipulating the Exception Context**:
   - Cause an access violation, and during the interception, manipulate the context to point to undetected code.
3. **Dual Views Protection**:
   - Hyperion uses dual views for memory, preventing race conditions and multi-threaded access manipulation.

### Conclusion

Hyperion is a robust anti-tamper tool using advanced user-mode techniques. It is not easily bypassed without deep Windows internals knowledge and reverse engineering expertise. Hyperion has successfully patched big executors like Krnl, Synapse X, and Script-Ware.

**Credits**:
- **Shade** for detailed bypass explanations.
- **Gogo** for detailed explanations and threads on Hyperion reversals.
- **@0AVX** for correcting information on INT3s.

Thank you for reading. I hope this helps to better understand Hyperion.