---
layout: page
title: comon
---

Table of contents:

- [Introduction](#introduction)
- [Loading the extension](#loading-the-extension)
- [Working with COM metadata](#working-with-com-metadata)
- [Tracing COM interactions](#tracing-com-interactions)
- [Choosing what and when to trace](#choosing-what-and-when-to-trace)
- [Errors and limitations](#errors-and-limitations)

## Introduction

**Comon** is a WinDbg extension that can help you trace COM interactions (COM class creations and interface querying). You may use it to investigate various COM issues and better understand application logic. During a debugging session, comon will record virtual table addresses (for the newly created COM objects) and allow you to query them or even set breakpoints on COM interface methods. If COM metadata is available (either in the registry or in a standalone TLB/DLL file), you may load it into comon, and it will automatically decode COM identifiers. Let's quickly walk through the comon functionalities.

COM objects used in this tutorial come from [my COM example project](https://github.com/lowleveldesign/protoss-com-example) (more information about it are available in [this blog post](https://lowleveldesign.org/2022/01/17/com-revisited/)).

## Loading the extension

To start using comon, you need to load it as any other extension:
```
.load comon
```
If a metadata file is already created (more about metadata later in the article), comon will load it. The next step is to attach comon to the process(es) we want. When debugging a single process, it is a matter of calling **!comon attach**. In multi-process debugging sessions, switch to the target process (for example, `|1s`) and run the **!comon attach** command. If you're debugging a 32-bit process with 64-bit WinDbg, ensure the effective architecture is correct (`.effmach` should return `x86`).

## Working with COM metadata

We need COM metadata to resolve CLSIDs and IIDs, identifiers of COM classes and interfaces. Without metadata, the comon output contains only raw GUIDs and may be hard to read. COM metadata is stored in an SQLite database, located in the user's temporary folder.

The primary command to work with metadata is **!cometa**. The subcommand **index** indexes COM registrations in the registry. Those include type libraries (the newest installed version), CLSIDs, and IIDs. The 64-bit version of the extension scans both 64-bit and 32-bit versions of the CLSID and Interfaces keys. If you provide a path to a TLB or DLL file to the **!cometa index** command, it will index it and add found metadata to the database. When indexing a DLL file, it must contain a type library as one of its resources. Type libraries are the best metadata sources, as they provide type names, methods, and parent types. With complete metadata for a given interface, you will be able to set breakpoints using its method names instead of ordinal numbers.

Comon also provides commands to query the indexed metadata and virtual table addresses. **!cometa showi** displays information about a given IID, and **!cometa showc** exhibits information about a given CLSID. Example output:

```
0:000> !cometa showi {C5F45CBC-4439-418C-A9F9-05AC67525E43}
Found: {C5F45CBC-4439-418C-A9F9-05AC67525E43} (INexus)

Methods:
- [0] QueryInterface
- [1] AddRef
- [2] Release
- [3] CreateUnit

Registered VTables for IID:
- Module: protoss (64-bit), CLSID: {F5353C58-CFD9-4204-8D92-D274C7578B53} (Protoss Nexus), VTable offset: 0x3b028
```

```
0:000> !cometa showc {F5353C58-CFD9-4204-8D92-D274C7578B53}
Found: {F5353C58-CFD9-4204-8D92-D274C7578B53} (Protoss Nexus)

Registered VTables for CLSID:
- module: protoss (64-bit), IID: {59644217-3E52-4202-BA49-F473590CC61A} (IGameObject), VTable offset: 0x3b058
- module: protoss (64-bit), IID: {C5F45CBC-4439-418C-A9F9-05AC67525E43} (INexus), VTable offset: 0x3b028
```

## Tracing COM interactions

After attaching, comon creates breakpoints on comon COM create methods. At the moment, those include methods: `CoCreateInstance`, `CoGetClassObject`, and `IClassFactory::CreateInstance`. This set does not cover all the possible ways to create COM class instances, but it should work in most cases. And you may always use the **!comon add** command to manually register a virtual table address. As I mentioned virtual tables, it is high time to understand how comon works and what its output contains. Whenever the target process hits the COM create method mentioned above, comon sets a one-time breakpoint on the method return. When the method successfully creates an instance of a COM object, comon will save its CLSID, IID, and address of the virtual table. It will also create a breakpoint on the `IUnknown::QueryInterface` method. Thanks to this breakpoint, we will be able to track all IIDs that a given COM instance resolves. Comon breakpoints are not visible through the `bl` command as comon uses a private copy of the `IDebugClient` and marks its breakpoints as private. I did it purposedly to avoid cluttering the breakpoint window or the `bl` output. If you want to list those private breakpoints for some reason, you may use the **!cobl** command. However, a more straightforward way to query virtual table addresses is through the cometa subcommands mentioned above. 

If you want to set a breakpoint on a specific COM interface method, you can use the **!cobp** command. The breakpoints set using this command will be public, and you may manage them using the regular WinDbg breakpoint commands (`bl`, `be`, `bd`, `bc`). The cobp command takes CLSID, IID, and a method name or number as its parameters. For example, to set a breakpoint on the `CreateUnit` method of the `INexus` interface, you may run:

```
!cobp F5353C58-CFD9-4204-8D92-D274C7578B53 C5F45CBC-4439-418C-A9F9-05AC67525E43 CreateUnit
```
If you don't have metadata indexed, you need to use an ordinal method number (`IUnknown` contains three methods, so the first method of an `INexus` interface has index 4):
```
!cobp F5353C58-CFD9-4204-8D92-D274C7578B53 C5F45CBC-4439-418C-A9F9-05AC67525E43 4
```

To stop comon monitoring of an active process, run the `!comon detach` command. The collected COM metadata will still be available, but comon will no longer log any details about COM calls.

## Choosing what and when to trace

As you can imagine, setting breakpoints on all `IUnknown::QueryInterface` methods in a COM-heavy application significantly impacts its performance. That's why comon provides ways to make its tracing less intrusive. To temporarily disable comon breakpoints, run the **!comon pause**. One-time breakpoints set on method returns will still trigger, but other breakpoints will be inactive, and comon won't create new breakpoints until you call **!comon resume**. The pause and resume subcommands are pretty drastic, so if you don't want to stop the tracing entirely but limit it to only specific classes (CLSIDs), the **!colog** filtering subcommands will help. You may either use inclusive (**!colog include**) or exclusive (**!colog exclude**) filters. For example, by calling `!colog include {F5353C58-CFD9-4204-8D92-D274C7578B53}`, you will make comon monitor `IUnknown::QueryInterface` methods only on the Nexus class. Adding more CLSIDs to the inclusive filter is possible by calling **!colog include** multiple times. If the inclusive filter is too limiting, you may consider using the exclusive filter. As its name suggests, it controls which `IUnknown::QueryInterface` breakpoints the monitor should skip.

## Errors and limitations

If **comon can't load a database file** with metadata (it always tries cometa.db3 in the user's temporary folder), it will use an in-memory database. Please make sure that there is no other application locking the cometa.db3 file. 

As mentioned above, comon currently hooks three COM class creation functions: `CoCreateInstance`, `CoGetClassObject`, and `IClassFactory::CreateInstance`. If the target application uses some other function, comon won't know about the created classes. To fix that, you may create a regular debugger breakpoint, read the virtual table after the object creation and add it to the comon internal database (`coadd`). If the create function comes from one of the system libraries, please [report it](https://github.com/lowleveldesign/comon/issues), and I will add it to comon.

