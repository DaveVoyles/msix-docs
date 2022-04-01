---
description: This article provides a deeper dive on how the Desktop Bridge works under the covers.
title: Understanding how packaged desktop apps run on Windows
ms.date: 01/30/2020
ms.topic: article
keywords: windows 10, uwp, msix
ms.assetid: a399fae9-122c-46c4-a1dc-a1a241e5547a
---

# Understanding how packaged desktop apps run on Windows

This article provides a deeper dive on what happens to files and registry entries when you create a Windows app package for your desktop application.

A key goal of a modern package is to separate application state from system state as much as possible while maintaining compatibility with other apps. Windows 10 accomplishes this by placing the application inside a MSIX package, and then detecting and redirecting some changes it makes to the file system and registry at runtime.

Packages that you create for your desktop application are desktop-only, full-trust applications and are not virtualized or sandboxed. This allows them to interact with other apps the same way classic desktop applications do.

## Installation

App packages are installed on a per user basis instead of system wide. App packages are installed under *C:\Program Files\WindowsApps\package_name*, with the executable titled *app_name.exe*. Each package folder contains a manifest (named AppxManifest.xml) that contains a special XML namespace for packaged apps. Inside that manifest file is an ```<EntryPoint>``` element, which references the full-trust app. When that application is launched, it does not run inside an app container, but instead it runs as the user as it normally would.

After deployment, package files are marked read-only and heavily locked down by the operating system. Windows prevents apps from launching if these files are tampered with.

## File system

The OS supports different levels of file system operations for packaged desktop applications, depending on the folder location.

### Optimized for your device

In order to avoid duplication of files to optimize for disk storage space and reduce the bandwidth needed when downloading files, the OS leverages single storage and hard linking of files. When a user downloads an MSIX package, the AppxManifest.xml is used to determine if the data contained with the package already exist on disk from an earlier package installation. If the same file exists in multiple MSIX packages, the OS stores the shared file on disk only once and create hard links from both packages to the shared file. Since files are downloaded in 64k blocks, even if a percentage of a file being downloaded exists on disk, only the increment that is different is downloaded. This reduces the bandwidth used for downloading.

### AppData operations on Windows 10, version 1903 and later

All newly created files and folders in the user's AppData folder (e.g., *C:\Users\user_name\AppData*) are written to a private per-user, per-app location but merged at runtime to appear in the real AppData location. This allows some degree of state separation for artifacts that are only used by the application itself, and this enables the system to clean up those files when the application is uninstalled. Modifications to existing files under the user's AppData folder is allowed to provide a higher degree of compatibility and interactivity between applications and the OS. This reduces filesystem “rot” because the OS is aware of every file or directory change made by an application. State separation also allows packaged desktop applications to pick up where a non-packaged version of the same application left off. Note that the OS does not support a virtual file system (VFS) folder for the user's AppData folder.

### AppData operations on Windows 10, version 1809 and earlier

All writes to the user's AppData folder (e.g., *C:\Users\user_name\AppData*), including create, delete, and update, are copied on write to a private per-user, per-app location. This creates the illusion that the packaged application is editing the real AppData when it is actually modifying a private copy. By redirecting writes this way, the system can track all file modifications made by the app. This allows the system to clean up those files when the application is uninstalled, thus reducing system "rot" and providing a better application removal experience for the user.

### Other folders

In addition to redirecting AppData, Windows' well-known folders (System32, Program Files (x86), etc) are dynamically merged with corresponding directories in the app package. Each package contains a folder named "VFS" at its root. Any reads of directories or files in the VFS directory are merged at runtime with their respective native counterparts. For example, an application could contain *C:\Program Files\WindowsApps\package_name\VFS\SystemX86\vc10.dll* as part of its app package, but the file would appear to be installed at *C:\Windows\System32\vc10.dll*.  This maintains compatibility with desktop applications that may expect files to live in non-package locations.

Writes to files/folders in the app package are not allowed. Writes to files and folders that are not part of the package are ignored by the OS and are allowed as long as the user has permission.

### Common operations

This short reference table shows common file system operations and how the OS handles them.

| Operation | Result | Example |
| --- | --- | --- |
| Read or enumerate a well-known Windows file or folder | A dynamic merge of *C:\Program Files\package_name\VFS\well_known_folder* with the local system counterpart. | Reading *C:\Windows\System32* returns the contents of *C:\Windows\System32* plus the contents of *C:\Program Files\WindowsApps\package_name\VFS\SystemX86*. |
| Write under AppData | **Windows 10, version 1903 and later:** New files and folders created under the following directories are redirected to a per-user, per-package private location:<ul><li>Local</li><li>Local\Microsoft</li><li>Roaming</li><li>Roaming\Microsoft</li><li>Roaming\Microsoft\Windows\Start Menu\Programs</li></ul>In response to a file open command, the OS will open the file from the per-user, per-package location first. If this location doesn't exist, the OS will attempt to open the file from the real AppData location. If the file is opened from the real AppData location, no virtualization for that file occurs. File deletes under AppData are allowed if user has permissions.<p/><p/>**Windows 10, version 1809 and earlier:** Copy-on-written to a per-user, per-app location. | AppData is typically *C:\Users\user_name\AppData*. |
| Write inside the package | Not allowed. The package is read-only. | Writes under *C:\Program Files\WindowsApps\package_name* are not allowed. |
| Writes outside the package | Allowed if the user has permissions. | A write to *C:\Windows\System32\foo.dll* is allowed if the package does not contain *C:\Program Files\WindowsApps\package_name\VFS\SystemX86\foo.dll* and the user has permissions. |

### Packaged VFS locations

The following table shows where files shipping as part of your package are overlaid on the system for the app. Your application will perceive these files to be in the listed system locations, when in fact they are in the redirected locations inside *C:\Program Files\WindowsApps\package_name\VFS*. The FOLDERID locations are from the [**KNOWNFOLDERID**](/windows/win32/shell/knownfolderid) constants.

System Location | Redirected Location (Under [PackageRoot]\VFS\) | Valid on architectures
 :--- | :--- | :---
FOLDERID_SystemX86 | SystemX86 | x86, amd64
FOLDERID_System | SystemX64 | amd64
FOLDERID_ProgramFilesX86 | ProgramFilesX86 | x86, amd6
FOLDERID_ProgramFilesX64 | ProgramFilesX64 | amd64
FOLDERID_ProgramFilesCommonX86 | ProgramFilesCommonX86 | x86, amd64
FOLDERID_ProgramFilesCommonX64 | ProgramFilesCommonX64 | amd64
FOLDERID_Windows | Windows | x86, amd64
FOLDERID_ProgramData | Common AppData | x86, amd64
FOLDERID_System\catroot | AppVSystem32Catroot | x86, amd64
FOLDERID_System\catroot2 | AppVSystem32Catroot2 | x86, amd64
FOLDERID_System\drivers\etc | AppVSystem32DriversEtc | x86, amd64
FOLDERID_System\driverstore | AppVSystem32Driverstore | x86, amd64
FOLDERID_System\logfiles | AppVSystem32Logfiles | x86, amd64
FOLDERID_System\spool | AppVSystem32Spool | x86, amd64

## Registry

App packages contain a registry.dat file, which serves as the logical equivalent of *HKLM\Software* in the real registry. At runtime, this virtual registry merges the contents of this hive into the native system hive to provide a singular view of both. For example, if registry.dat contains a single key "Foo", then a read of *HKLM\Software* at runtime will also appear to contain "Foo" (in addition to all the native system keys).

Although MSIX packages include HKLM and HKCU keys, they are treated differently. Only keys under *HKLM\Software* are part of the package; keys under *HKCU* or other parts of the registry are not. Writes to keys or values in the package are not allowed. Writes to keys or values not part of the package are allowed as long as the user has permission.

All writes under HKCU are copy-on-written to a private per-user, per-app location. Traditionally, uninstallers are unable to clean *HKEY_CURRENT_USER* because the registry data for logged out users is unmounted and unavailable.

All writes are kept during package upgrade and only deleted when the application is removed entirely.

### Common operations

This short reference table shows common registry operations and how the OS handles them.

Operation | Result | Example
:--- | :--- | :---
Read or enumerate *HKLM\Software* | A dynamic merge of the package hive with the local system counterpart. | If registry.dat contains a single key "Foo," at runtime a read of *HKLM\Software* will show the contents of both *HKLM\Software* plus *HKLM\Software\Foo*.
Writes under HKCU | Copy-on-written to a per-user, per-app private location. | The same as AppData for files.
Writes inside the package. | Not allowed. The package is read-only. | Writes under *HKLM\Software* are not allowed if a corresponding key/value exist in the package hive.
Writes outside the package | Ignored by the OS. Allowed if the user has permissions. | Writes under *HKLM\Software* are allowed as long as a corresponding key/value does not exist in the package hive and the user has the correct access permissions.

## Uninstallation

When a package is uninstalled by the user, all files and folders located under *C:\Program Files\WindowsApps\package_name* are removed, as well as any redirected writes to AppData or the registry that were captured during the packaging process.
