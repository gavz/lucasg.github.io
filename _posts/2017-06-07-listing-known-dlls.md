---
layout: post
title: "Listing KnownDlls"
date: 2017-06-07
---

The `KnownDlls` is a nifty little trick used by Windows  to speed up the loading of "default" system shared libraries, using a `COW` `(Copy on Write)` mechanism for fast mapping in memory. 

One question though, is it possible to list all the dll under `KnownDlls` ?

<!--more-->

The Windows dll search order is fairly complicated and slow since it must scan the following folders (in the default search) : 

```
* The directory from which the application loaded.
* The system directory. Use the GetSystemDirectory function to get the path of this directory.
* The 16-bit system directory. There is no function that obtains the path of this directory, but it is searched.
* The Windows directory. Use the GetWindowsDirectory function to get the path of this directory.
* The current directory.
* The directories that are listed in the PATH environment variable. ( Note that this does not include the per-application path specified by the App Paths registry key. The App Paths key is not used when computing the DLL search path.)
```

the `KnownDlls` mechanism has been introduced in 2006 by [`KB 164501`](https://support.microsoft.com/en-nz/help/164501/info-windows-nt-2000-xp-uses-knowndlls-registry-entry-to-find-dlls) in order to speed up dll loading. It leverage the fact that most `Win32` applications load the same set of system libraries (`ntdll.dll`, `kernel32.dll`, etc.) : instead of scanning folders on disk, those particular dlls can be mapped in memory and copied into newly created processes, thus greatly improving process creation times.

`KnownDlls` can easily spotted by looking at several processes loaded module lists : they are always mapped at the same virtual base address accross processes (which has security implications ...).


In order to list all the dll considered `KnownDlls`, I had to read some documentation. As always, there is a pertinent blog post on the msdn network :[ https://blogs.msdn.microsoft.com/larryosterman/2004/07/19/what-are-known-dlls-anyway](https://blogs.msdn.microsoft.com/larryosterman/2004/07/19/what-are-known-dlls-anyway).

According to the author, if some dlls are "statically" listed as `KnownDlls` in the registry key `HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\SessionManager\KnownDLLs`, it does not return the entire list of librairies since all the dlls loaded by a `KnownDll` is also a `KnownDll`. In order to "dynamically" list all the `KnownDlls`, we need to list all the sections under `\KnownDlls` (for 64-bit dlls) and/or `\KnownDlls32` (for wow64-bit dlls).


# Listing files from a directory handle

Problem : `\KnownDlls` is a "special" directory since it's not really present on disk's FS. 

The first implication is `FindFirstFile`/`FindNextFile` won't work, since it rely on `\KnownDlls` being a traditionnal folder. 
Nevertheless we can get a directory handle on `\KnownDlls` via `CreateFile` (via the `FILE_FLAG_BACKUP_SEMANTICS` flag). However the available API to manipulate directory handles are quite poor :

	* BackupRead 	
	* BackupSeek 	
	* BackupWrite 	
	* GetFileInformationByHandle 	
	* GetFileSize 	
	* GetFileTime 	
	* GetFileType 	
	* ReadDirectoryChangesW 	
	* SetFileTime

(source : [https://msdn.microsoft.com/en-us/library/windows/desktop/aa365258(v=vs.85).aspx](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365258(v=vs.85).aspx))

The `Backup*` apis are useless in our case, as well as `GetFile*` and `SetFileTime`. Only `GetFileInformationByHandle` and `ReadDirectoryChangesW` seem remotely useful. Unfortunately, `GetFileInformationByHandle` does not return useful infos to us and `ReadDirectoryChangesW` only notify changes inside the directory, not the current state of it.

However there is another way to list elements living under a directory handle, using `NtQueryDirectoryObject`. `NtQueryDirectoryObject` is a **meh** documented native api that ```retrieves information about the specified directory object``` which does not say much about what it does. Fortunately, some people on the Internet hinted this NT API was used by `Winobj` to read the content of global objects such as `Devices`, ``BaseNamedObjects` or `KnownDlls`.

The `NtQueryDirectoryObject` API is a bit clunky to use (just read [`rewolf`'s post about it](http://blog.rewolf.pl/blog/?p=1273)) since you need to provide a buffer that will be written by increments of `OBJECT_DIRECTORY_INFORMATION` data structures. If the buffer size is too small, you need to keep track of a `Context` parameter that store which entries already has been returned. It looks like to me like a old-style generator or the way Linux's char devices handle `echo/cat` commands.


Anyway, I've found two implementations which enumerate objects under a DirectoryObjects:

* [C] `PhEnumDirectoryObjects` in [`processhacker2\ProcessHacker2\phlib\native.c`](https://github.com/processhacker2/processhacker2/blob/master/phlib/native.c) . It's used by ProcessHacker2's `ExtraPlugins\ObjectManagerPlugin` which is a `winobj`-like plugin.

* [C#] [https://randomsourcecode.wordpress.com/2015/03/14/enumerating-deviceobjects-from-user-mode](https://randomsourcecode.wordpress.com/2015/03/14/enumerating-deviceobjects-from-user-mode)



Here my quick and dirty implementation in `C` :

{%highlight C%}
typedef NTSTATUS (WINAPI *_NtQueryDirectoryObject_T)(
	_In_      HANDLE  DirectoryHandle,
	_Out_opt_ PVOID   Buffer,
	_In_      ULONG   Length,
	_In_      BOOLEAN ReturnSingleEntry,
	_In_      BOOLEAN RestartScan,
	_Inout_   PULONG  Context,
	_Out_opt_ PULONG  ReturnLength
);

typedef struct _UNICODE_STRING
{
	USHORT Length;
	USHORT MaximumLength;
	_Field_size_bytes_part_(MaximumLength, Length) PWCH Buffer;
} UNICODE_STRING, *PUNICODE_STRING;

typedef struct _OBJECT_DIRECTORY_INFORMATION
{
	UNICODE_STRING Name;
	UNICODE_STRING TypeName;
} OBJECT_DIRECTORY_INFORMATION, *POBJECT_DIRECTORY_INFORMATION;

typedef struct _OBJECT_ATTRIBUTES
{
	ULONG Length;
	HANDLE RootDirectory;
	PUNICODE_STRING ObjectName;
	ULONG Attributes;
	PVOID SecurityDescriptor; // PSECURITY_DESCRIPTOR;
	PVOID SecurityQualityOfService; // PSECURITY_QUALITY_OF_SERVICE
} OBJECT_ATTRIBUTES, *POBJECT_ATTRIBUTES;

#define DIRECTORY_QUERY 0x0001
#define STATUS_MORE_ENTRIES              ((NTSTATUS)0x00000105L)
#define NT_SUCCESS(Status) (((NTSTATUS)(Status)) >= 0)

#define InitializeObjectAttributes(p, n, a, r, s) { \
    (p)->Length = sizeof(OBJECT_ATTRIBUTES); \
    (p)->RootDirectory = r; \
    (p)->Attributes = a; \
    (p)->ObjectName = n; \
    (p)->SecurityDescriptor = s; \
    (p)->SecurityQualityOfService = NULL; \
    }

int main(int argc, char *argv[])
{
	HANDLE KnownDllDir;
	FILE_FULL_DIR_INFO DirInfo;
	FILE_ID_BOTH_DIR_INFO ExtDirInfo;
	BY_HANDLE_FILE_INFORMATION lpFileInformation;
	_NtQueryDirectoryObject_T NtQueryDirectoryObject_I = (_NtQueryDirectoryObject_T)GetProcAddress(GetModuleHandle(L"ntdll.dll"), "NtQueryDirectoryObject");
	POBJECT_DIRECTORY_INFORMATION buffer;
	NTSTATUS status;

	OBJECT_ATTRIBUTES oa;
	UNICODE_STRING name = {
		.Length = 20,
		.MaximumLength = 20,
		.Buffer = L"\\KnownDlls"
	};

	InitializeObjectAttributes(
		&oa,
		&name,
		0,
		NULL,
		NULL
	);

	status = NtOpenDirectoryObject(
		&KnownDllDir,
		DIRECTORY_QUERY,
		&oa
	);


	if (!NT_SUCCESS(status))
		return -1;

	BOOLEAN firstTime = TRUE;
	ULONG Context = 0;
	size_t bufferSize = 0x200;
	size_t ReturnLength;

	buffer = malloc(bufferSize);
	
	while(TRUE)
	{
		
		while ((status = NtQueryDirectoryObject(
			KnownDllDir,
			buffer,
			bufferSize,
			FALSE,
			firstTime,
			&Context,
			&ReturnLength
		)) == STATUS_MORE_ENTRIES)
		{
			// Check if we have at least one entry. If not, we'll double the buffer size and try
			// again.
			if (buffer[0].Name.Buffer)
				break;

			free(buffer);
			bufferSize *= 2;
			buffer = malloc(bufferSize);
		}

		size_t i = 0;

		while (TRUE)
		{
			POBJECT_DIRECTORY_INFORMATION info;
			

			info = &buffer[i];

			if (!info->Name.Buffer)
				break;


			if (0 == wcscmp(L"Section", info->TypeName.Buffer))
				printf("Knwown dll : %ws \n", (wchar_t*) info->Name.Buffer);

			i++;
		}

		if (status != STATUS_MORE_ENTRIES)
			break;

		firstTime = FALSE;
	}

	CloseHandle(KnownDllDir);
}
{%endhighlight C%}


Anyway, there is a "guy" that keep track of `KnownDlls` across Windows versions : [https://windowssucks.wordpress.com/knowndlls/](https://windowssucks.wordpress.com/knowndlls/).

