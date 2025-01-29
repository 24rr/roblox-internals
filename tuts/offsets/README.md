# Roblox Offsets

Written by **[@trypt.amine](https://github.com/tryptamine)**.

## GetTaskScheduler:

1. Strings -> "Can't initialize the TaskScheduler before flags have been loaded" -> XRef (this is the TaskScheduler initializer)
2. Open XRefs of the function containing said string, from the bottom open the 4th call reference
3. If it looks like this:

```c
 v5 = sub_7FF6458E5D30(v4); <-- TaskScheduler initializer
 dword_7FF64824D928 = 0;
 TaskScheduler_singleton = v5; <-- singleton!
```
4. Found it!

## TaskSchedulerDeobfJobs:

1. Strings -> "TaskScheduler::Job:" -> Open the only call reference
2. Navigate to the top of the function, locate the call that uses Arg1 + OFFSET as an argument (usually the 3rd call) 

```c
v2 = sub_7FF641E0BB44();
if (v2)
    sub_7FF641E0B534(v2);
v3 = a1 + 384;
v38 = v3;
sub_7FF641632430(v3); <-- this one here it is!
```
3. Profit! :money_mouth:

## rbx_getscheduler:

1. TaskSchedulerDeobfJobs -> Xrefs -> Find a call that doesn't use "this" or a global (it's the singleton instance)
2. It should look like this:

```c
v2 = rbx_getscheduler(); <— here it is!
v17 = v2 + 176;
v18 = 0;
v3 = sub_7FF764F8D850(v2 + 176);
if (v3)
    sub_7FF764F8D240(v3);
v18 = 1;
v4 = v2 + 384;
v19 = v4;
TaskSchedulerDeobfJobs((__int64*)v4);
```