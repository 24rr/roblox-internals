# Overview
As you probably know by now, Hyperion crashes upon detecting a blacklisted process like Cheat Engine, Extreme Injector, and similar tools. Below is a detailed breakdown of the blacklisted process detection methods employed by Hyperion.

## 1) Hyperion Illegal Process Detection
- Roblox crashes upon the startup of tools like Cheat Engine and other reverse engineering software because they are identified as blacklisted processes.
Detection Methods:
```json
{

  "HYPERION_DETECT_BLACKLISTED_PROCESS": [

    "NtOpenKey",

    "NtQueryKey KeyValueBasicInformation",

    "NtQuerySystemInformation",

    "NtQuerySystemInformation",

    "NtQuerySystemInformation",

    "NtUserFindWindowEx",

    "NtOpenProcess",

    "NtQueryInformationProcess",

    "NtQuerySystemInformation",

    "NtQueryObject",

    "NtOpenDirectoryObject",

    "NtQueryDirectoryObject",

    "NtQueryVirtualMemory",

    "NtQueryAttributesFile",

    "NtSetInformationThread"

  ],

  "threads": [

    "ntdll.dll+0x35580"

  ]
}
```

There are multiple ways to bypass these checks. One approach is hooking NtOpenProcess and NtQueryDirectoryObject.
Another method, which I personally dislike, involves suspending any thread with the start address ntdll.dll+0x35580 using tools like [System Informer](https://systeminformer.com/). However, avoid terminating these threads, as it leads to an endless cycle of new calls since they will continuously be added to the stack.

## 2) Triggering Blacklisted Process Detection and Termination of RobloxPlayerBeta ->
Some of you may recognize this Application Popup, which states:Third-party software is interfering with Roblox. If you're using software for exploiting or reverse-engineering, you'll need to uninstall it."
This popup, executed by "csrss.exe," is what we're going to analyze.

Every Faulting Process & "Error Messages" are always logged in the preinstalled application Event Viewer
which makes our life easier since we do not have to code our own Faulting Application Logger
These events, such as those related to "RobloxPlayerBeta.exe,"
Are logged in the Windows Logs folder. By simply expanding the dropdown and selecting the System view, we can easily inspect them.
All Roblox error logs are associated with Event ID 26. Additionally, these application popups are executed by csrss.exe.

## 3 Hyperion's Ce_Lua detection
Cheat Engine integrates with Roblox's Lua API by hooking into memory allocation.
- Hyperion silently detects this integration and sends analytics to Roblox, which can lead to a delayed ban. Lua usage is instantly flagged, as this system was intentionally designed to disrupt scripts relying on Lua and Cheat Engine.

## Intercepted Calls Before Termination Due to Blacklisted Process Detection
Moving Down
you'll notice that RobloxCrashHandler.exe writes to the path Roblox/logs/crashes/attachments
which contains crash logged information, including:
- PlaceID
- Session Duration
- PlayerID
- SessionReport, which is always labeled as AppStatusCrash.

## Additional Notes
Roblox actively checks if Process Hacker is running and utilizes it for analytics purposes. This behavior is expected to be reversed in future analysis.
