**Unmasking the Hyperion Emulator: A Deep Dive into the Latest Roblox Exploit**

---

AWP Analysis made by Gogo

Hello everyone, 0xNemi here. Today, I’m diving into the latest buzz in the Roblox exploiting scene—a new executor that claims to not only be undetected but also capable of emulating Hyperion, Roblox’s advanced anti-tamper system. As someone who’s spent countless hours reverse-engineering Hyperion, I was both skeptical and intrigued. The developers even offered me a free license to test their claims, so I decided to put their executor under the microscope. Here’s what I found.

---

### **The Setup**
The executor in question replaces the standard Roblox launcher with a custom loader called `loader-proxy-rs.exe`, written in Rust. This loader replaces the original Hyperion DLL with an unsigned, smaller module that supposedly contains their Hyperion emulator. The UI and loader didn’t interest me much—they’re just for bootstrapping. The real meat was in the module that replaced Hyperion.

When I loaded this module into IDA, it crashed immediately. Turns out, they were exploiting a known IDA issue (CVE-2024-44083) to hinder static analysis. Clever, but ultimately futile—I bypassed it by removing the jump chain and redirecting the entry point to the original code.

---

### **The Emulator**
The entry point had some irrelevant code for updating the working directory, likely due to the proxy loader messing it up. After that, the code entered a virtualized section, which appeared to use Themida. At this point, I switched to runtime analysis using my hypervisor.

The module performed extensive system scans for reverse engineering tools, a clear sign they were serious about anti-debugging. Interestingly, they copied Hyperion’s hypervisor detection mechanisms, but the real Hyperion wasn’t running. This initially supported their claim of emulating Hyperion.

The module then added a VEH (Vectored Exception Handler) and hooked two critical functions: `LdrInitializeThunk` and `KiUserExceptionDispatcher`. The VEH code was surprisingly unobfuscated, which was a red flag. It handled Roblox-to-Hyperion messages, a critical part of Hyperion’s functionality. This code was essential for emulating Hyperion, as the original DLL wasn’t loaded to handle these messages.

---

### **Networking and Yara Handling**
The executor’s handling of Yara packets was particularly impressive. They loaded Hyperion without resolving any imports, copied the allocation, passed it to Roblox’s Yara scan function, and forwarded the result to `send_packet`. This was a clever way to mimic Hyperion’s behavior without actually running it.

They also recreated Hyperion’s packet encryption, including the recent ChaCha20 constant transformation. This level of detail suggested a deep understanding of Hyperion’s internals.

---

### **The EMD.bin File**
One interesting detail was the `EMD.bin` file written to the Roblox directory. This file contained decrypted code from Roblox, which the executor used to override the encrypted code in the original binary before passing control to Roblox. They also handled imports dynamically, streaming data from the server in JSON format. Here’s a sample of the JSON structure:

```json
{
    "code_rva": 4096,
    "code_size": 64188416,
    "imports": [
        {
            "export": "RegOpenKeyExA",
            "mod": "KERNELBASE.dll",
            "rva": 64196608
        },
        {
            "export": "RegQueryValueExA",
            "mod": "KERNELBASE.dll",
            "rva": 64196616
        },
        {
            "export": "RegCloseKey",
            "mod": "KERNELBASE.dll",
            "rva": 64196624
        },
        ..........
    ],
    "original_entrypoint_rva": 109682692,
    "real_entrypoint_rva": 60832452,
    "scanner_rva": 6962240,
    "send_packet_rva": 6961984,
    "tls1_rva": 60833424,
    "tls2_rva": 60833544
}
```

---

### **Security Measures**
The executor’s security was a mixed bag. The entire code section was encrypted using a homemade algorithm, not a known packer. They also re-encrypted the code during runtime, similar to Hyperion’s anti-tamper methods. However, the executor module itself wasn’t heavily secured, as it relied on the emulator module (`RobloxPlayerBeta.dll`) for protection.

They used lazy importer, `xorstr`, and manual syscalls for critical checks. Their manual syscall implementation was particularly interesting—it wrote the syscall instruction inside a `.svm` section, used a secondary context with its own stack, and switched to it using `IRET` with the Trap Flag (TF) enabled. This made tracing the syscall’s origin more challenging.

---

### **Conclusion**
After thorough analysis, I can confirm that their Hyperion emulator is real. Whether it perfectly replicates Hyperion’s networking or leaves detectable traces is harder to determine without deeper analysis. However, their understanding of Hyperion’s internals is impressive, especially compared to other executors that are often detected by the same mechanisms.

As for the Lua environment accusations, I didn’t focus on that aspect. If they’re faking it after achieving such a sophisticated emulator, it would be disappointing, as the Lua environment is relatively trivial compared to the emulator.

---

### **Final Thoughts**
This executor is a fascinating piece of work, but I’m not endorsing it or any other P2C. My goal is to share knowledge and help the community make informed decisions. What I dislike most is incompetent developers challenging others without proof, relying solely on community trust. Bold claims should always be backed by detailed, reproducible analysis—just as I’ve done here.

If you’re interested in the technical details, check out the images below for a closer look at the code and structures:

- [Entry Point](https://user-assets.v3rm.net/attachments/16/16116-0679443f985c19a81e989a79af82c114.jpg?hash=Wm7aroV2pF)
- [Encrypted Code Section](https://user-assets.v3rm.net/attachments/16/16117-d3d8eeef2579e75258ddb74fd465a526.jpg?hash=ZvKQc4XoFS)
- [VEH Handler](https://user-assets.v3rm.net/attachments/16/16118-5b8be5d0b71b6ef0acffe6d91473ac2f.jpg?hash=djjv7swYIh)
- [Yara Handling](https://user-assets.v3rm.net/attachments/16/16122-73e1911fa2a527eaec4c159ddb6d6a5b.jpg?hash=yYjAPMemum)
- [Packet Encryption](https://user-assets.v3rm.net/attachments/16/16121-33687d6deba7b28c108742c83ddbd875.jpg?hash=RjUBnvGp0P)
- [Code Encryption](https://v3rm.net/attachments/code_enc-png.16104/)

Stay curious, stay skeptical, and always verify claims with evidence. Until next time! 🚀