# How to Code a Pattern Scanner for Renderview in Roblox

In this tutorial, I'll show you how to code a pattern scanner to obtain the `RenderView` in Roblox without needing the task scheduler offsets. This method works whether Roblox is just being opened or is already running. With an optimized and multi-threaded pattern scanner, it takes between 20-250ms to obtain the `RenderView`.

## Example in Golang

Below is an example implementation in Golang:

```go
package main

import (
    "fmt"
    "sync"
    "time"
    "unsafe"
    "sort"
    "golang.org/x/sys/windows"
)

// Define your instance and memory structures
type RobloxInstances struct {
    Pid       int64
    ExeName   string
    Mem       *Memory
    Instances Instances
    Offsets   OffsetsDataPlayer
}

type Instances struct {
    RenderView uintptr
    RobloxBase uint64
}

type Memory struct {
    RobloxBase uintptr
    ProcessHandle uintptr
}

type OffsetsDataPlayer struct {
    // Define your offsets here
}

type Instance struct {
    Address uintptr
    Parent  *RobloxInstances
}

func NewInstance(address uintptr, parent *RobloxInstances) *Instance {
    return &Instance{Address: address, Parent: parent}
}

func (i *Instance) FindFirstChild(name string, recursive bool) *Instance {
    // Implement your logic to find the first child
    return i
}

func (i *Instance) GetChildren() []*Instance {
    // Implement your logic to get children
    return []*Instance{}
}

func (i *Instance) String() string {
    return fmt.Sprintf("Instance at 0x%X", i.Address)
}

// Pattern scanning functions
func findAllPatterns(data, pattern []byte, baseAddress uintptr) []uintptr {
    var results []uintptr
    patternLength := len(pattern)
    dataLength := len(data)

    for i := 0; i <= dataLength-patternLength; i++ {
        found := true
        for j := 0; j < patternLength; j++ {
            if pattern[j] != 0x00 && pattern[j] != data[i+j] {
                found = false
                break
            }
        }
        if found {
            results = append(results, baseAddress+uintptr(i))
        }
    }

    return results
}

func findPattern(data, pattern []byte, baseAddress uintptr) uintptr {
    patternLength := len(pattern)
    dataLength := len(data)

    for i := 0; i <= dataLength-patternLength; i++ {
        found := true
        for j := 0; j < patternLength; j++ {
            if pattern[j] != 0x00 && pattern[j] != data[i+j] {
                found = false
                break
            }
        }
        if found {
            return baseAddress + uintptr(i)
        }
    }

    return 0
}

// Memory region structure
type MemoryReg struct {
    base  uintptr
    size  uintptr
    state uint32
    prot  uint32
    alloc uint32
}

// AOBSCANALL function
func (p *Memory) AOBSCANALL(AOB_HexArray string, xreturn_multiple bool, stop_at_value int) ([]uintptr, error) {
    pattern := []byte(AOB_HexArray) // Convert hex string to byte array
    var results []uintptr

    var regions []MemoryReg
    var mbi windows.MemoryBasicInformation
    address := uintptr(0)

    for {
        err := windows.VirtualQueryEx(windows.Handle(p.ProcessHandle), address, &mbi, unsafe.Sizeof(mbi))
        if err != nil {
            break
        }
        if mbi.State == 0x1000 && mbi.Protect == 0x04 && mbi.AllocationProtect == 0x04 {
            regions = append(regions, MemoryReg{
                base:  address,
                size:  mbi.RegionSize,
                state: mbi.State,
                prot:  mbi.Protect,
                alloc: mbi.AllocationProtect,
            })
        }

        address += mbi.RegionSize
    }

    if len(regions) == 0 {
        return nil, fmt.Errorf("no readable memory regions found")
    }

    resultsCh := make(chan []uintptr, len(regions))
    errCh := make(chan error, len(regions))

    var wg sync.WaitGroup
    wg.Add(len(regions))

    for _, region := range regions {
        go func(r MemoryReg) {
            defer wg.Done()
            data, err := p.ReadMemory(r.base, r.size)
            if err != nil {
                errCh <- err
                resultsCh <- nil
                return
            }
            if xreturn_multiple {
                localResults := findAllPatterns(data, pattern, r.base)
                resultsCh <- localResults
            } else {
                singleResult := findPattern(data, pattern, r.base)
                if singleResult != 0 {
                    resultsCh <- []uintptr{singleResult}
                } else {
                    resultsCh <- nil
                }
            }
        }(region)
    }

    go func() {
        wg.Wait()
        close(resultsCh)
        close(errCh)
    }()

    for res := range resultsCh {
        if len(res) > 0 {
            results = append(results, res...)
            if !xreturn_multiple {
                break
            }
            if stop_at_value > 0 && len(results) >= stop_at_value {
                break
            }
        }
    }

    sort.Slice(results, func(i, j int) bool { return results[i] < results[j] })

    return results, nil
}

func main() {
    // Example usage
    mem := &Memory{ProcessHandle: 0x1234} // Replace with actual process handle
    datamodel, _ := mem.AOBSCANALL("RenderJob(EarlyRendering;", false, 1)

    New := &RobloxInstances{
        Pid:     1234, // Replace with actual PID
        ExeName: "RobloxPlayerBeta.exe",
        Mem:     mem,
        Instances: Instances{
            RobloxBase: uint64(mem.RobloxBase),
        },
        Offsets: OffsetsDataPlayer{},
    }

    var DM *Instance

    const (
        RenderView = 0x1E8
    )

    rv, _ := mem.ReadPointer(datamodel[0] + uintptr(RenderView))

    const (
        Fake = uintptr(0x118)
        Real = uintptr(0x1A8)
    )

    FakeDatamodelContainer, _ := mem.ReadPointer(rv + Fake)
    realdm, _ := mem.ReadPointer(FakeDatamodelContainer + Real)
    DM = NewInstance(realdm, New)

    fmt.Println(DM.String(), mem.RobloxBase)

    Plr := DM.FindFirstChild("Players", false)

    for {
        for _, inst := range Plr.GetChildren() {
            fmt.Println(inst.String())
        }

        time.Sleep(time.Millisecond * 50)
    }
}
```

## Explanation

1. **Pattern Scanning**: The `findAllPatterns` and `findPattern` functions are used to scan memory for a specific byte pattern. This is crucial for locating the `RenderJob` string in memory.

2. **Memory Regions**: The `AOBSCANALL` function scans all readable memory regions in the target process to find the pattern. It uses multi-threading to speed up the scanning process.

3. **Renderview and Datamodel**: Once the `RenderJob` pattern is found, you add `0x1E8` to get the `RenderView` address. You then read the address to a pointer type to get the original `RenderView` address. This address is cached for future use.

4. **Fake and Real Datamodel**: The `RenderView` address is used to find the `FakeDatamodelContainer` by adding `0x118`. The real datamodel is then obtained by adding `0x1A8` to the `FakeDatamodelContainer`.

5. **Caching**: The `RenderView` address is cached so that you don't need to rescan for it during the session. This improves performance and reduces the risk of detection.

## Important Notes

- **Offsets**: The `FakeDatamodelContainer` location has offsets to multiple jobs. You can explore these offsets in Cheat Engine for more details.
- **Performance**: Ensure your pattern scanner is optimized and multi-threaded to achieve the best performance.
