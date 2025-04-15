---
layout: single
title:  Path to Privilege: How Armoury Crate's File Deletion Flaw Creates System-Wide Risk
date: 2025-4-15
classes: wide
header:
  teaser: 
--- 
### Overview

A critical vulnerability exists in ASUS Armoury Crate, a system management software that controls RGB lighting, device performance, and driver updates on ASUS hardware. While the main application runs with user privileges, it communicates with a backend service running as SYSTEM to execute various tasks. The service fails to properly validate and sanitize user-controlled file paths in recursive delete operations. This allows an unprivileged local attacker to delete arbitrary files or directories on the system, potentially leading to system integrity loss, denial of service, or **Local Privilege Escalation (LPE)** through tampering with security-critical files.

Preinstalled on many ASUS motherboards and laptops, this software is deeply integrated into the system, making it difficult to remove without accessing the BIOS. Everyone assumes bloatware and preinstalled programs are merely annoying, not dangerous—but they absolutely can be.
https://rog.asus.com/us/content/armoury-crate/
I was added to the ASUS hall of fame for this discovery https://www.asus.com/content/asus-product-security-advisory/. Although I didn't discover this through reverse engineering the code in ILSpy, exploring the vulnerability's exact location and examining its implementation in C# was fascinating. Patch diffing and analyzing the implemented fix was also particularly interesting.

![](/assets/images/HOF.png)

![](/assets/images/HOF2.png)

---
### Discovery: 
One of the least interesting parts of this vulnerability to me is the actual exploitation as it's basically taking a well known technique (`FolderContentsDeleteToFolderDelete`) with some small modifications for ease of use to accomplish LPE. More interesting to me was investigating how this vulnerability originated and examining the fix that was implemented.

Here's the snippet of code that enables exploitation:
![](/assets/images/suslpe1)
The unprivileged Armoury Crate utility communicates with the Armoury Crate service via TCP (not shown) to execute the 'Delete Content' command. When provided with a directory path, it recursively deletes files from that location.

This wouldn't typically be a problem in directories where normal users lack write permissions. However, the vulnerability arises because malicious actors can create mount points and symlinks within accessible directories, redirecting the deletion process to target files of their choice elsewhere on the system.

The service executes recursive file deletion operations on user-supplied directory paths WITHOUT:
1. Validating paths against safe, pre-defined locations
2. Checking for reparse points before privileged operations
3. Operating with least-privilege principles

There are several ways to fix this vulnerability (non-exhaustive list below), as multiple conditions must be true for exploitation to occur:
- Impersonate the user initiating the delete operation, as is done for the openAWE command located directly above where the vulnerable pattern exists. This would ensure any file deletion attempt would be limited to files a normal user could delete anyway.
- Implement proper permission controls for the directory if possible, following least privilege principles. This would mitigate the vulnerability by preventing normal users from placing mount points or symlinks in the path.
- Avoid blind recursive deletion by targeting specific known file names and extensions. The system could track which files are created when a wallpaper is generated and only delete those specific files. It's possible to set filters for directory enumeration to target specific files or folders—I've observed this behavior in other software that successfully blocked similar exploitation attempts.
- Check whether each file is a reparse point before deletion to prevent following symbolic links to unauthorized locations.


So how is a recursive delete actually turned into an ARBITRARY file delete? 
sidenote: on 24h2 it's harder to get an arb file delete to system if it's not a recursive delete. I want to expand on this in a later post although it could likely be condensed into a tweet.
### Proof of Concept (PoC):
All credit to the very talented Abdelhamid Naceri ([halov](https://twitter.com/KLINIX5)) who discovered this. 
Steps from: https://www.zerodayinitiative.com/blog/2022/3/16/abusing-arbitrary-file-deletes-to-escalate-privilege-and-other-great-tricks

1. Create a subfolder, `temp\folder1`.
2. Create a file, `temp\folder1\file1.txt`.
3. Set an oplock on `temp\folder1\file1.txt`.
4. Wait for the vulnerable process to enumerate the contents of `temp\folder1` and try to delete the file `file1.txt` it finds there. This will trigger the oplock.
5. When the oplock triggers, perform the following in the callback:  
    a. Move `file1.txt` elsewhere, so that `temp\folder1` is empty and can be deleted. We move `file1.txt` as opposed to just deleting it because deleting it would require us to first release the oplock. This way, we maintain the oplock so that the vulnerable process continues to wait, while we perform the next step.  
    b. Recreate `temp\folder1` as a junction to the ‘\RPC Control`folder of the object namespace. c. Create a symlink at`\RPC Control\file1.txt`pointing to`C:\Config.Msi::$INDEX_ALLOCATION`.
6. When the callback completes, the oplock is released and the vulnerable process continues execution. The delete of `file1.txt` becomes a delete of `C:\Config.Msi`.
### Video PoC:
[https://youtu.be/GYPHj2B_-oE](https://youtu.be/GYPHj2B_-oE)

---
### Checking out the fix!
I reported this vulnerability to ASUS and they got back to me exceptionally quickly and got a patch out by the end of the month!

![](/assets/images/suslpe2)
The deletion is now passed to AuraWallpaperUserSessionHelper.exe with the 'DeleteAWEContent' flag as a normal user. The previously privileged deletion is now performed by the user who initiated the delete operation. Epic!
![](/assets/images/suslpe3)

There were also several new functions added to the service itself that facilitate more secure file handling, which did not exist in the previous version.
![](/assets/images/suslpe4)
The program is no longer vulnerable to arbitrary file deletion resulting from insecure file handling. Update your Armoury Crate!"

Since this vulnerability could be triggered unauthenticated through TCP once on a host, it's possible to exploit it without any GUI access. I didn't invest time in developing this approach, but the format of these requests is quite simple and could easily be replicated to trigger the delete and subsequent privilege escalation without a GUI. Users typically don't update this software promptly, so vulnerable versions are likely to remain unpatched for some time. This implementation is left as an exercise for the reader.

---
### Affected Version

- Aura Wallpaper Service V 2.0.15.0
- Aura Wallpaper HTML V 2.0.15.0

---
### Timeline:
3/5/2025 - Disclosed vulnerability to ASUS PSIRT
3/10/2025 - Non duplicatable by ASUS due to not enough details being provided. I responded with more details...
3/13/2055 - "We had forwarded your message and RD is working on it. Once we have feedback from them, we will get back to you soonest.
3/18/2025 - "...According to our SW team, the new version is estimated to be released at the end of March. When it’s released, we will let you know as well."
3/27/2025 - New patch has been released! 
	"According to our SW team, new patch had been released and it will be auto-updated when you execute Armoury Crate.
	You may also go to **Armoury Crate** **à** **Settings** **à** **Update Center** **à** **Check for updates** to manually update below and check whether the vulnerbility is fixed or not.
	Aura Wallpaper Service V 2.0.15.0
	Aura Wallpaper HTML V 2.0.15.0"




