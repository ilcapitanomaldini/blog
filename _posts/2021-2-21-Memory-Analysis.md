---
layout: post
title: Memory analysis for iOS apps
---

# Memory analysis for iOS apps

This document explores some steps that can be undertaken and tools that could be leveraged when solving memory issues or optimizing for it.
 
##Introduction
 
Before we begin, let’s look at some aspects of memory usage and relevant keywords around it. While generally we would expect memory to be managed by the system using reference counting mechanism, Objective-C files can disable it on a per-file basis using _-fno-objc-arc_ compiler flag to manually manage it. Manually memory-managed files need more careful analysis. So, if your application uses this flag then beware!

If we do depend on reference counting then there’s still ways to get memory issues: 
**1. Memory leaks**: When we have allocated memory for some objects that we no longer need and caused by,
**2. Retain cycles**: Two or more objects retain strong references to each other meaning that the reference count does not go down to 0 when we would expect it.
**3. Dangling references**: Deallocated memory that we still point to/hold a reference variable to.
 
When talking about dynamic memory allocations, there are 2 types: 
**1. Heap allocations**: All reference type objects created by the application are allocated on the heap.
**2. VM(Virtual Memory) allocations**: It is the VM space allocated by OS to the application and handled by it but even so, it is generally in response to something the application is doing. Examples include: memory-mapped files, CALayer backing store, CGImage data, etc. There’s no direct control over this area but it must still be analysed.

Also, when dealing with the Instruments tool we can have **Persistent**(memory that stayed around for the time range selected) or **Transient**(memory that was created and destroyed too in the time range selected) memory which can be filtered respectively.


### Things to watch out for

 - Leaks: An obvious choice but not always the silver bullet as a fix.
 - Persistent memory: While a high usage of memory shouldn't be problematic if the application demands it (high-res images, videos, webViews, etc.), it is a concern if it keeps on increasing even when parts of it may no longer be needed.
 - Transient memory: This isn't an issue in most cases but keep a lookout if optimizing a module. If there's high churn in a specific time range then spikes generated may push usage over the limit or it may even be the cause of slowdowns. 
 - Repeated calls or object allocations that could have been avoided.


### Instruments

There are 3 instrument tools we can leverage: 
1. Review the memory report in Xcode directly.
2. Allocations tool in Instruments: Mark generations before and after using the feature/module under test. Make comparisons and draw conclusions based on the memory allocations observed within that time period.
3. Leaks tool in Instruments: Use this to identify the leaks in the time period marked above. Make use of the call tree and the generated graphs resolve them.

 - Note: Ensure NSZombies option is turned off if looking for accurate memory measurements. Keep it on when looking for leaks or related issues.

 
### Memory graph, or vmmap analysis on memgraph generated

The debug memory graph is an Xcode tool that helps to visualize the object references and their inter-dependencies. This is the quickest way to get some actionable observations if you already have a suspect. It might be beneficial for some projects to include analysing this graph before submitting a feature. Relatively easy to figure out which objects are alive past their lifetime and its cause.
Find the memory graph options here:

![Memgraph Bottom Icon](/images/Memgraph_BottomIcon.png)

![Memgraph Side Panel Icon](/images/Memgraph_SideIcon.png) 

If some more detailed analysis is required, then export the memory graph generated so that more insights can be gleaned from it.

![Memgraph Export Option](/images/Memgraph_Export_Option.png)	

You can turn on allocation stack traces, by toggling the Malloc Stack box in the Diagnostics area of your scheme’s Run settings. With it, the memory allocations in the memory graph would be associated with methods.

Malloc zones are a variable-size range of virtual memory from which the memory system can allocate blocks. Some example offenders of VM usage can be: 
1. MALLOC_NANO 
2. __LINKEDIT (refers to symbol tables that are saved in a file)
3. __TEXT (It contains executable code and constants)
4. mapped file (This region contains file contents that are frequently accessed and are mapped to the virtual memory in order to enable faster access).

 - Note: Toggle the Malloc Stack option in Run settings off when exporting to get more accurate readings.

###### Commands reference

 - vmmap --summary name.memgraph
 - vmmap --verbose name.memgraph | grep "module/address"
 - leaks --traceTree 0x[address] name.memgraph
 - heap name.memgraph -sortBySize
 - malloc_history name.memgraph --fullStacks 0x[address]


### ips file analysis (using symbolication) 

ips files are crash reports generated from iOS(go to the Organizer, to Library Device Logs, and Import). For memory related issues, jetsam files are generated which are described in the section below but still the ips files might have a linked root cause and would provide some more insights.

###### Helpful links:

 - [Apple doc for crash report diagnosis](https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs)
 - [SO: Symbolicate](https://stackoverflow.com/questions/25855389/how-to-symbolicate-crash-log-xcode)
 - [Apple doc for adding symbols](https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/adding_identifiable_symbol_names_to_a_crash_report)

###### Command reference for manual symbolication:

 - atos -arch arm64 -o App_name.app.dSYM/Contents/Resources/DWARF/App_name -l 0x10443c000 0x000000010515728c
 - ./symbolicatecrash -v App_name-2020-11-13-152836.ips App_name.app.dSYM


### Jetsam file analysis 

Jetsam file generated on a memory-related crash might indicate, for example, that the largest process in memory was the app with the reason being “per-process-limit“. It means that the process crossed the resident memory limit imposed by the system on all apps. Crossing this limit makes the process eligible for termination. Limit can be say ~1.5GB of resident memory for the device in question. 

 - [Developer doc for jetsam events](https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/identifying_high-memory_use_with_jetsam_event_reports)
   - Note: The example mentions rpages value of 92,802 which was eerily close to observations
 - [Device types and their hard limits](https://stackoverflow.com/a/15200855)


### Memory Logs
 
Suggested use of the following to get a sense of memory usage from  logs:

```
func memUse() -> (resident: UInt64, physical: UInt64) {
        var taskInfo = task_vm_info_data_t()
        var count = mach_msg_type_number_t(MemoryLayout<task_vm_info>.size) / 4
        let result: kern_return_t = withUnsafeMutablePointer(to: &taskInfo) {
            $0.withMemoryRebound(to: integer_t.self, capacity: 1) {
                task_info(mach_task_self_, task_flavor_t(TASK_VM_INFO), $0, &count)
            }
        }
        
        var physical: UInt64 = 0
        var resident: UInt64 = 0

        if result == KERN_SUCCESS {
            physical = UInt64(taskInfo.phys_footprint)
            resident = UInt64(taskInfo.resident_size)
        }
        
        return (resident, physical)
    }
```

and log it by:

```
//res and phys are stored vars
let memUsage = memUse()
let resGrowth = usage.resident - (res <= usage.resident ? res : usage.resident)
res = usage.resident
let physGrowth = usage.physical - (phys <= usage.physical ? phys : usage.physical)
phys = usage.physical
debugPrint("Resident: \(usage.resident), with growth: \(resGrowth), Physical: \(usage.physical), with growth: \(physGrowth)")
``` 
 
 


### Other references

 - [Apple Docs: About the Virtual Memory System](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/ManagingMemory/Articles/AboutMemory.html)
 - [Breakdown on VM usage and meaning](https://stackoverflow.com/a/35833857)
 - [Discussion about transient spikes possibly being root cause](https://developer.apple.com/forums/thread/54336)
   - [Another regarding spikes and speed of allocations](https://stackoverflow.com/a/16319798)


### TO-DO:

 - [] Add concrete examples
 - [] Expand sections with more description where necessary
 - [] Add a section for custom instrument
 - [] Update proper references 
 
 
### Extra: 
![DAGs](/images/DAGS.png)
_Directed Acyclic Graphs! Learn to love them._