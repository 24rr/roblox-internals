Bypassing this check is pretty easy; I'm surprised Roblox hasn't patched it yet. We can do it by patching the WinVerifyTrust function that is part of the Wintrust.dll DLL, you can use this method in conjunction with the SetWindowsHookEx function to inject into ROBLOX. While this isn't the only way to bypass the DLL signature checks, it is the simplest.

```c
void HookWindowsVerify() {

    DWORD PID = GetPID("RobloxPlayerBeta.exe"); // get the process id of roblox
    HANDLE ProcessHandle = OpenProcess(PROCESS_ALL_ACCESS, FALSE, PID);

    HINSTANCE LibHandle = LoadLibraryA("Wintrust.dll"); // load the DLL with the "WinVerifyTrust" function in it.
    FARPROC FunctionAdress = GetProcAddress(LibHandle, "WinVerifyTrust"); // get the address of the function we want to patch.

    CHAR patch[5] = { 0xC3, 0x90, 0x90, 0x90, 0x90 }; // create a patch (1st instruction is a RET others are NOPs)

    SIZE_T bytesWritten;
    WriteProcessMemory(ProcessHandle, (LPVOID)FunctionAdress, patch, 5, &bytesWritten); // write the patch into the process.


    // cleanUp
    FreeLibrary(LibHandle);
    CloseHandle(ProcessHandle);


    // now you can use your shitty SetWindowsHookEx injection
}
```

No problem skids, see ya.
kthxbye.