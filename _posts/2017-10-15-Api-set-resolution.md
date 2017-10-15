---
layout: post
title: "Api set resolution"
date: 2017-10-15
---

[Windows API Sets schema](https://msdn.microsoft.com/en-us/library/windows/desktop/hh802935(v=vs.85).aspx) is a weird dll redirection mechanism  Microsoft introduced in Win7 (and perfected in Win8.1) but no one really know why it's useful. What's more there is little to none official documentation on the subject (and it's not even up to date) which makes me think MS don't really want to officially support the schema.

All I know (before writing this post) is to solve a missing api min-win, you usually rely on copying the whole redist folder when deploying a statically compiled binary and calling it a day : 

{% include image.html url="/assets/apiset-copy.PNG" description="Can you find which application does that ?" %}

Fortunately, [third-party research]({{page.url}}#References) covers pretty much everything you need to know the subject. However, I had to write an api set resolver for the [Dependencies application](https://lucasg.github.io/Dependencies) I've been working on this year. Following in this post is how the NT loader resolve an api set library.

<!--more-->


Api sets dll are "virtual libraries" actually implementing only contract APIs which will be resolved by the NT loader. According to [Ionescu's Esoteric Hooks presentation](https://github.com/ionescu007/HookingNirvana/raw/master/Esoteric%20Hooks.pdf) the official aim is to decouple 'API' features (heap allocations, strings manipulations, etc.) from subsystem implementations (enumerating processes, registering services, etc.). I'm pretty sure this is also another step into getting rid of the undying cockroach that is the Win32 subsystem in Windows desktops. Anyway, api set schema is probably what made [UWP apps](https://docs.microsoft.com/en-us/windows/uwp/get-started/whats-a-uwp) possible :

![that's actually from the MS patent on the subject](/assets/apiset-patent.PNG)

As said previously, the api set schema (and its evolution along Windows versions) has already been reversed and publicly documented, so I'll just do a quick recap on it's actual implementation.

`%Windir%\System32\apisetschema.dll` is the dll implementing the contract resolution between virtual api-min-win dlls and host libraries in its `.apiset` section :

![.apiset section data](/assets/apiset-section.PNG)

This section is actually present in every process via the PEB (probably using a COW mechanism ). :

![.apiset section data address in the PEB](/assets/apiset-peb.PNG)

The `PEB.ApiSetMap` points to a `API_SET_NAMESPACE_V6` (not an official denomination) on Windows 10 : 

{%highlight C %}

typedef struct {
  ULONG Version;     // v2 on Windows 7, v4 on Windows 8.1  and v6 on Windows 10
  ULONG Size;        // apiset map size (usually the .apiset section virtual size)
  ULONG Flags;       // according to Geoff Chappell,  tells if the map is sealed or not.
  ULONG Count;       // hash table entry count
  ULONG EntryOffset; // Offset to the api set entries values
  ULONG HashOffset;  // Offset to the api set entries hash indexes
  ULONG HashFactor;  // multiplier to use when computing hash 
} API_SET_NAMESPACE;

{%endhighlight C %}

The api set schema namespace is basically a hash table with values being the following structure : 

{%highlight C %}

// Hash table index (just an optimization "trick")
typedef struct {
  ULONG Hash;
  ULONG Index;
} API_SET_HASH_ENTRY;

// Hash table value
typedef struct {
  ULONG Flags;        // sealed flag in bit 0
  ULONG NameOffset;   // Offset to the ApiSet library name PWCHAR (e.g. "api-ms-win-core-job-l2-1-1")
  ULONG NameLength;   // Ignored
  ULONG HashedLength; // Apiset library name length
  ULONG ValueOffset;  // Offset the list of hosts library implement the apiset contract (points to API_SET_VALUE_ENTRY array)
  ULONG ValueCount;   // Number of hosts libraries 
} API_SET_NAMESPACE_ENTRY;

// Host Library entry
typedef struct {
  ULONG Flags;        // sealed flag in bit 0
  ULONG NameOffset;   // Offset to the ApiSet library name PWCHAR (e.g. "api-ms-win-core-job-l2-1-1")
  ULONG NameLength;   // Apiset library name length
  ULONG ValueOffset;  // Offset to the Host library name PWCHAR (e.g. "ucrtbase.dll")
  ULONG ValueLength;  // Host library name length
} API_SET_VALUE_ENTRY;

{% endhighlight C %}


The ApiSet resolution is implemented in 3 non exported functions in the `ntdll` (although there are public symbols present) :

* `ApiSetResolveToHost`  for the "high level" resolution (take a `UNICODE_STRING` name and fill out another `UNICODE_STRING` structure)
* `ApiSetpSearchForApiSet` which implement the ApiSetMap hash table retrieval
* `ApiSetpSearchForApiSetHost` which can discriminate dll when an apiset library has several "hosts" libraries (not a frequent case).


`ApiSetpSearchForApiSet` is a pretty traditionnal hash table getter. Since hash buckets may collide, there is an additional check on the apiset library name returned by the hash table : 

{% highlight C %}

  PAPI_SET_NAMESPACE_ENTRY 
  __fastcall ApiSetpSearchForApiSet(
    _In_ PAPI_SET_NAMESPACE ApiNamespace,
    _In_ PWCHAR ApiNameToResolve, 
    _In_ uint16_t ApiNameToResolveSize
  )
  {
    __int16 _ApiNameToResolveSize; // si@1
    ULONG HashKey; // er9@1
    const WCHAR *_ApiNameToResolve; // rbp@1
    API_SET_NAMESPACE *ApiNamespacePtr; // r10@1
    PWCHAR pApiNameToResolve; // r11@1
    __int64 _ApiNameToResolveCount; // rbx@2
    WCHAR ApiNameToResolveCurChar; // dx@3
    API_SET_NAMESPACE_ENTRY *FoundEntry; // rbx@6
    int HashCounter; // er8@6
    int ApiSetEntryCount; // ecx@6
    int HashIndex; // edx@7
    signed __int64 HashOffset; // r11@7

    _ApiNameToResolveSize = ApiNameToResolveSize;
    HashKey = 0;
    _ApiNameToResolve = ApiNameToResolve;
    ApiNamespacePtr = ApiNamespace;
    pApiNameToResolve = ApiNameToResolve;

    FoundEntry = NULL;
    HashCounter = 0;
    ApiSetEntryCount = ApiNamespace->Count - 1;

    if (ApiSetEntryCount < 0)
      return NULL;

    if (!ApiNameToResolveSize)
      return NULL;
    
    // HashKey = Hash(ApiNameToResolve.ToLower())
    _ApiNameToResolveCount = (uint16_t) ApiNameToResolveSize;
    do
    {
      ApiNameToResolveCurChar = *pApiNameToResolve;
      if ((unsigned __int16)(*pApiNameToResolve - 'A') <= 0x19u) // that's BAD normalization !
        ApiNameToResolveCurChar += ' ';
      ++pApiNameToResolve;

      HashKey = HashKey * ApiNamespace->HashFactor + ApiNameToResolveCurChar;
      
      --_ApiNameToResolveCount;
    } while (_ApiNameToResolveCount);
  

    // Looking for matching hash key
    while (1)
    {
      HashIndex = (ApiSetEntryCount + HashCounter) >> 1;
      HashOffset = ApiNamespacePtr->HashOffset + sizeof(uintptr_t) * HashIndex;
      if (HashKey < ((ULONG *)((uintptr_t) ApiNamespacePtr + HashOffset))[0])
      {
        ApiSetEntryCount = HashIndex - 1;
        goto CHECK_COUNTERS;
      }
      if (HashKey <= ((ULONG *)((uintptr_t) ApiNamespacePtr + HashOffset))[0])
        break;

      HashCounter = HashIndex + 1;

    CHECK_COUNTERS:
      if (HashCounter > ApiSetEntryCount)
        return NULL;
    }
    
    // Get the corresponding hash bucket value
    FoundEntry = (API_SET_NAMESPACE_ENTRY *)((char *)ApiNamespacePtr
      + sizeof(API_SET_NAMESPACE_ENTRY) * *(ULONG *)((char *)&ApiNamespacePtr->Size + HashOffset)
      + ApiNamespacePtr->EntryOffset);

    if (!FoundEntry)
      return NULL;

    // Check the returned hash entry actually correspond to the given ApiSet name
    if(0 == RtlCompareUnicodeStrings(
      /* _In_ PWCHAR */ _ApiNameToResolve,
      /* _In_ SHORT  */ _ApiNameToResolveSize,
      /* _In_ PWCHAR */ ((uintptr_t)ApiNamespacePtr + FoundEntry->NameOffset),
      /* _In_ SHORT  */ FoundEntry->HashedLength >> 1, // FoundEntry->HashedLength / sizeof(WCHAR)
      TRUE              // Ignore case
    )) {
      return FoundEntry;
    }


    return NULL;
  }

{%endhighlight C %}


Since the returned entry may reference multiple hosts library (apparently that's a feature MS devs needed) there is an additional function to return the exact host dll based on a "ParentName" : `ApiSetpSearchForApiSetHost`. Honestly, I always set the `ParentName` to `NULL` and get correct results so there must be a "default" result.

{% highlight C %}

  PAPI_SET_VALUE_ENTRY 
  __stdcall ApiSetpSearchForApiSetHost(
    _In_ PAPI_SET_NAMESPACE_ENTRY Entry, 
    _In_ PWCHAR *ParentName,
    _In_ SHORT ParentNameLen,
    _In_ PAPI_SET_NAMESPACE ApiNamespace
  )
  {
    __int64 _EntryValueOffset; // r12@1
    int Counter; // ebp@1
    API_SET_NAMESPACE *_ApiNamespacePtr; // r15@1
    int EntryAliasCount; // ebx@1
    PWCHAR *_ApiToResolveSource; // r10@1
    API_SET_VALUE_ENTRY *FoundEntry; // rdi@1
    SHORT _ApiToResolveLen; // r13@2
    int EntryAliasIndex; // esi@3
    API_SET_VALUE_ENTRY *AliasEntry; // r14@3
    int _result; // eax@3
    PWCHAR *_ApiToResolve; // [sp+68h] [bp+10h]@1

    _ApiToResolveSource = ParentName;
    
    _ApiNamespacePtr = ApiNamespace;
    _EntryValueOffset = Entry->ValueOffset;
    FoundEntry = (API_SET_VALUE_ENTRY *)((uintptr_t)ApiNamespace + _EntryValueOffset);

    // Unique host library entry : bail immediately
    EntryAliasCount = Entry->ValueCount - 1;
    if (EntryAliasCount == 0)
      return FoundEntry;

    Counter = 1;
    _ApiToResolve = ParentName;
    _ApiToResolveLen = ParentNameLen;
    do
    {
      EntryAliasIndex = (EntryAliasCount + Counter) >> 1;
      AliasEntry = (API_SET_VALUE_ENTRY *)((char *)_ApiNamespacePtr
        + sizeof(API_SET_VALUE_ENTRY) * EntryAliasIndex
        + _EntryValueOffset);

      // Compare API_SET_VALUE_ENTRY.NameOffset to ParentName
      _result = RtlCompareUnicodeStrings(
        /* _In_ PWCHAR */ _ApiToResolveSource,
        /* _In_ SHORT  */ _ApiToResolveLen,
        /* _In_ PWCHAR */ ((uintptr_t)_ApiNamespacePtr + AliasEntry->NameOffset),
        /* _In_ SHORT  */ AliasEntry->NameLength >> 1,
        TRUE // IgnoreCase 
      );

      if (_result < 0)
      {
        EntryAliasCount = EntryAliasIndex - 1;
      }
      else
      {
        if (_result == 0)
        {
          return (API_SET_VALUE_ENTRY *)((char *)_ApiNamespacePtr
            + sizeof(API_SET_VALUE_ENTRY) * ((EntryAliasCount + Counter) >> 1)
            + _EntryValueOffset);
        }

        Counter = EntryAliasIndex + 1;
      }

      _ApiToResolveSource = _ApiToResolve;
    } while (Counter <= EntryAliasCount);

    return FoundEntry;
  }
{% endhighlight C %}


`ApiSetResolveToHost` is the function wrapping the other previous two in order to "hide" the hash table implementation details from the point a view of a third-party developer (being here a MS dev since none of this mechanism is officially accessible). The only singular points are :

* `ApiSetResolveToHost` checks the apiset library name is actually prefixed by `"api-"` or `"ext-"`
* the apiset library name is truncated before being fed to `ApiSetpSearchForApiSet` : it get rid of the `".dll"` extension (which is a just an application hint) and everything after the last hyphen. That's actually understandable : after the last hyphen is the "build" version, and supporting a strict comparison of apiset library name would make the ApiSet Map size explode. 

 
{% highlight C %}
  
  const uint64_t API_ = (uint64_t)0x2D004900500041; // L"api-"
  const uint64_t EXT_ = (uint64_t)0x2D005400580045; // L"ext-";

  NTSTATUS 
  __fastcall ApiSetResolveToHost(
    _In_ PAPI_SET_NAMESPACE ApiNamespace,
    _In_ PUNICODE_STRING ApiToResolve, 
    _In_ PUNICODE_STRING ParentName, 
    _Out_ PBOOLEAN Resolved, 
    _Out_ PUNICODE_STRING Output
  )
  {
    __int64 ApiNamespacePtr; // rdi@1
    char IsResolved; // bl@1
    PBOOLEAN pIsResolved; // r15@1
    UNICODE_STRING *_ParentName; // r14@1
    unsigned __int16 ApiSetNameBufferSize; // cx@1
    wchar_t *ApiSetNameBuffer; // rdx@2
    uint64_t ApiSetNameBufferPrefix; // rax@2
    NTSTATUS Status; // rax@4
    unsigned int ApiSetNameWithoutExtensionSize; // eax@5
    uintptr_t pApiSetNamePtr; // rcx@5
    __int16 ApiSetNameWithoutExtensionWordCount; // ax@8
    API_SET_NAMESPACE_ENTRY *ResolvedEntry; // rax@9
    API_SET_VALUE_ENTRY *HostLibraryEntry; // rcx@12

    ApiNamespacePtr = (__int64)ApiNamespace;
    IsResolved = 0;
    pIsResolved = Resolved;
    _ParentName = ParentName;
    Output->Length = 0;
    Output->Buffer = NULL;
    ApiSetNameBufferSize = ApiToResolve->Length;

    if (ApiToResolve->Length >= 8u)
    {
      // --------------------------
      // Check library name starts with "api-" or "ext-"
      ApiSetNameBuffer = ApiToResolve->Buffer;
      ApiSetNameBufferPrefix = ((uint64_t*) ApiSetNameBuffer)[0] & 0xFFFFFFDFFFDFFFDF;
      if (ApiSetNameBufferPrefix == API_ || ApiSetNameBufferPrefix == EXT_)
      {


        // ------------------------------
        // Compute word count of apiset library name without the dll suffix
        // E.g. : 
        //     api-ms-win-core-apiquery-l1-1-0.dll -> len(api-ms-win-core-apiquery-l1-1)
        // ------------------------------
        ApiSetNameWithoutExtensionSize = ApiSetNameBufferSize;
        pApiSetNamePtr = ((uintptr_t)ApiSetNameBuffer) + ApiSetNameBufferSize;
        do
        {
          if (ApiSetNameWithoutExtensionSize <= 1)
            break;
          ApiSetNameWithoutExtensionSize -= sizeof(wchar_t);
          pApiSetNamePtr -= sizeof(wchar_t);

        } while (((wchar_t *)pApiSetNamePtr)[0] != '-');
        ApiSetNameWithoutExtensionWordCount = (uint16_t) ApiSetNameWithoutExtensionSize >> 1;
        // --------------------------

        if (ApiSetNameWithoutExtensionWordCount)
        {
          ResolvedEntry = ApiSetpSearchForApiSet(
            (API_SET_NAMESPACE *)ApiNamespacePtr,
            ApiSetNameBuffer,
            ApiSetNameWithoutExtensionWordCount);
          if (ResolvedEntry)
          {
            if (_ParentName && ResolvedEntry->ValueCount > 1)
            {
              HostLibraryEntry = (API_SET_VALUE_ENTRY *)ApiSetpSearchForApiSetHost(
                ResolvedEntry,
                (PWCHAR *)_ParentName->Buffer,
                _ParentName->Length >> 1,
                (API_SET_NAMESPACE *)ApiNamespacePtr);
              goto WRITING_RESOLVED_API;
            }
            if (ResolvedEntry->ValueCount > 0)
            {
              HostLibraryEntry = (API_SET_VALUE_ENTRY *)(ApiNamespacePtr + ResolvedEntry->ValueOffset);
            WRITING_RESOLVED_API:
              IsResolved = 1;
              Output->Buffer = (wchar_t *)(ApiNamespacePtr + HostLibraryEntry->ValueOffset);
              Output->MaximumLength = (SHORT) HostLibraryEntry->ValueLength;
              Output->Length = (SHORT) HostLibraryEntry->ValueLength;
              goto EPILOGUE;
            }
          }
        }
      }
    }
  EPILOGUE:
    Status = STATUS_SUCCESS;
    *pIsResolved = IsResolved;
    return Status;
  }

{% endhighlight C %}

<br/>

Anyway I've uploaded a gist of an apiset library resolver : [https://gist.github.com/lucasg/9aa464b95b4b7344cb0cddbdb4214b25](https://gist.github.com/lucasg/9aa464b95b4b7344cb0cddbdb4214b25).

{% highlight C %}

int wmain(int argc, wchar_t* argv[])
{
  if (argc < 2)
  {
    wprintf(L"ApiSetLookup : test for api set resolution.\n");
    return 0;
  }

  
  // Unit testing : this may not be true on your machine (that's kinda the point of the api set schema).
  API_SET_UNIT_TEST(L"api-ms-win-crt-runtime-l1-1-0.dll", L"ucrtbase.dll");
  API_SET_UNIT_TEST(L"api-ms-win-crt-math-l1-1-0.dll", L"ucrtbase.dll");
  API_SET_UNIT_TEST(L"api-ms-win-crt-stdio-l1-1-0.dll", L"ucrtbase.dll");
  API_SET_UNIT_TEST(L"api-ms-win-core-heap-l1-1-0.dll", L"kernelbase.dll");
  API_SET_UNIT_TEST(L"api-ms-win-core-job-l1-1-0.dll", L"kernelbase.dll");
  API_SET_UNIT_TEST(L"api-ms-win-core-job-l2-1-1.dll", L"kernel32.dll");
  API_SET_UNIT_TEST(L"api-ms-win-core-registry-private-l1-1-0.dll", L"advapi32.dll");
  API_SET_UNIT_TEST(L"api-ms-win-downlevel-ole32-l1-1-1.dll", L"combase.dll");
  API_SET_UNIT_TEST(L"api-ms-win-eventing-consumer-l1-1-1.dll", L"sechost.dll");
  API_SET_UNIT_TEST(L"ext-ms-onecore-appdefaults-l1-1-0.dll", L"windows.storage.dll");
  API_SET_UNIT_TEST(L"ext-ms-win-wer-wct-l1-1-0.dll", L"wer.dll");

  wchar_t *ApiSetLibraryName = argv[1];
  UNICODE_STRING HostApi = { 0 };
  if (ResolveApiSetLibrary(ApiSetLibraryName, &HostApi))
  {

    // HostApi.Buffer is not NULL terminated (probably to save some precious bytes since it's COW in every process)
    wchar_t HostLibraryName[MAX_PATH];
    _snwprintf_s(HostLibraryName, _countof(HostLibraryName), HostApi.Length >> 1, L"%s", HostApi.Buffer);


    wprintf(L"[!] Api set library resolved : %s -> %s\n", ApiSetLibraryName, HostLibraryName);
  }
  else
  {
    wprintf(L"[x] Could not resolve Api set library : %s.\n", ApiSetLibraryName);
  }

  return 0;
}

{% endhighlight C %}

That should help everyone that have a "missing" api min win import. It's also present in my [Dependencies tool](https://github.com/lucasg/Dependencies):

![apiset-in-dependencies](/assets/apiset-in-dependencies.PNG)

## <a name="References"></a>  References

* [Official documentation](https://msdn.microsoft.com/en-us/library/windows/desktop/hh802935(v=vs.85).aspx)
* [https://blog.quarkslab.com/runtime-dll-name-resolution-apisetschema-part-i.html](https://blog.quarkslab.com/runtime-dll-name-resolution-apisetschema-part-i.html)
* [https://blog.quarkslab.com/runtime-dll-name-resolution-apisetschema-part-ii.html](https://blog.quarkslab.com/runtime-dll-name-resolution-apisetschema-part-ii.html)
* [http://www.geoffchappell.com/studies/windows/win32/apisetschema/index.htm](http://www.geoffchappell.com/studies/windows/win32/apisetschema/index.htm)
* [https://ofekshilon.com/2016/03/27/on-api-ms-win-xxxxx-dll-and-other-dependency-walker-glitches/](https://ofekshilon.com/2016/03/27/on-api-ms-win-xxxxx-dll-and-other-dependency-walker-glitches/)
* [https://2016.zeronights.ru/wp-content/uploads/2016/12/You%E2%80%99re-Off-the-Hook.pdf](https://2016.zeronights.ru/wp-content/uploads/2016/12/You%E2%80%99re-Off-the-Hook.pdf)
* [Alex Ionescu mention api set in it's "Esoteric Hooks" presentation](https://github.com/ionescu007/HookingNirvana/raw/master/Esoteric%20Hooks.pdf)
* [Windows Internals 7th tools](https://github.com/zodiacon/WindowsInternals/blob/master/APISetMap/ApiSet.h)

## <a name="apisetmap"></a>  010 Template for parsing the ApiSet map


{% highlight c %}

//------------------------------------------------
//--- 010 Editor v8.0 Binary Template
//
//      File: 
//   Authors: 
//   Version: 
//   Purpose: 
//  Category: 
// File Mask: 
//  ID Bytes: 
//   History: 
//------------------------------------------------

// Api namespace header
typedef struct {
  ULONG Version;
  ULONG Size <format=hex>; 
  ULONG Flags;
  ULONG Count <format=hex>;
  ULONG EntryOffset <format=hex>;
  ULONG HashOffset <format=hex>;
  ULONG HashFactor <format=hex>;
} API_SET_NAMESPACE;

typedef struct {
  ULONG Hash;
  ULONG Index;
} API_SET_HASH_ENTRY;

typedef struct {
  ULONG Flags;
  ULONG NameOffset;
  ULONG NameLength;
  ULONG HashedLength;
  ULONG ValueOffset;
  ULONG ValueCount;
} API_SET_NAMESPACE_ENTRY;

typedef struct {
  ULONG Flags;
  ULONG NameOffset;
  ULONG NameLength;
  ULONG ValueOffset;
  ULONG ValueLength;
} API_SET_VALUE_ENTRY;


typedef struct {
    for (i=0; i<ApiSetMap.Count; i++) {
        API_SET_NAMESPACE_ENTRY Entry;
    }
} API_SET_NAMESPACE_ENTRIES;

typedef struct {
    for (i=0; i<ApiSetMap.Count; i++) {
        API_SET_HASH_ENTRY HashEntry;
    }
} API_SET_NAMESPACE_HASH_ENTRIES;

typedef struct (API_SET_VALUE_ENTRY &ValueEntry) {

    local int StartAddress = FTell();
    local int ValueLength = ValueEntry.ValueLength;

    CHAR Value[ValueEntry.ValueLength + 2];
} API_SET_ENTRY_VALUE;

typedef struct (API_SET_NAMESPACE_ENTRY& Entry) {
    local int ValueCount = Entry.ValueCount;
    local int NameLength = Entry.NameLength;
    local int StartAddress = FTell();
    local int HasValues = false;

    CHAR Name[Entry.NameLength + 2];

    FSeek(StartAddr + Entry.ValueOffset);

    API_SET_VALUE_ENTRY ValueEntry;
    FSeek(StartAddr + ValueEntry.ValueOffset);
    
    if (ValueEntry.ValueOffset)
    {
        HasValues = true;
        for (j = 0; j < Entry.ValueCount;j++)
        {      
            API_SET_ENTRY_VALUE ValueName(ValueEntry);
        }
    }
    
    if (i < ApiSetMap.Count - 1){
        FSeek(StartAddr + Entries.Entry[i+1].NameOffset);
    }
    else {
        FSeek(StartAddr + ApiSetMap.Size);
    }

} API_SET_ENTRY_PARSED <read=ReadApiSetEntry>;

string ReadApiSetEntry(API_SET_ENTRY_PARSED & ParsedEntry)
{
    local string sApiSetName = ReadWString(ParsedEntry.StartAddress, ParsedEntry.NameLength);
    
    if (ParsedEntry.HasValues) {
        local string sApiSetValue = ReadWString(ParsedEntry.ValueName[0].StartAddress, ParsedEntry.ValueName[0].ValueLength/2);
        return  sApiSetName + " -> " + sApiSetValue; 
    }

    return sApiSetName;
}

// main entry point
LittleEndian();
Printf("Parse api set schema Begin.\n");
local int StartAddr = FTell();
local int i =0;
local int j =0;

API_SET_NAMESPACE ApiSetMap <fgcolor=cGreen>;
FSeek(StartAddr + ApiSetMap.EntryOffset);

// Enumerate API_SET_NAMESPACE_ENTRY entries
API_SET_NAMESPACE_ENTRIES Entries  <fgcolor=cPurple>;

// Traverse API_SET_NAMESPACE_ENTRY entries and retrieve
// corresponding API_SET_VALUE_ENTRY entry in order to dump
// api-set and hosts library names.
for (i=0; i<ApiSetMap.Count; i++) {
    
    FSeek(StartAddr + Entries.Entry[i].NameOffset);
    API_SET_ENTRY_PARSED ParsedEntry(Entries.Entry[i]) <fgcolor=cBlue>;
}

// Enumerate API_SET_HASH_ENTRY entries
FSeek(StartAddr + ApiSetMap.HashOffset);
API_SET_NAMESPACE_HASH_ENTRIES HashEntries <fgcolor=cRed>;

Printf("Parse api set schema End.\n");

{% endhighlight c %}