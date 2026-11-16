
>[!tldr] In this Very Easy Sherlock, players will go through artefacts and logs from the Domain controller as well as endpoint artefacts from where Kerberoast attack activity was sourced. Players will work through what to look for to identify kerberoasting attack activity and to avoid false positives due to the complexity of Active Directory.

 *I will be using the Windows pwnbox provided by HTB*

---
1. `Analyzing Domain Controller Security Logs, can you confirm the UTC date & time when the kerberoasting activity occurred?`
	- First, I opened the Domain Controller logs in `Event Viewer`
>[!hint] In Security Logs, Filter for Event ID 4769. Now Look for any event where the service name is NOT( krbtgt or ends with $ (For e.g DC01$ ) ). The ticket type should be 0x17 which is for RC4 type encryption. The failure code should be 0x0. The event that matches all the above conditions is the event detailing information about the kerberoasting attack activity.
- I decided to write a `XML` filter:
```shell
<QueryList>
	<Query Id="0" Path="file://C:\Users\htb-ep-131823\Downloads\Triage\Domain Controller\SECURITY-DC.evtx">
	    <Select Path="file://C:\Users\htb-ep-131823\Downloads\Triage\Domain Controller\SECURITY-DC.evtx">
		    *[
		    System[(EventID=4769)]
			    and
			EventData[Data[@Name="ServiceName"]!="DC01$"]
		    ]
	    </Select>
	</Query>
</QueryList>
```
- With the filter applied there were 3 events left, though, applying what i know from the hint there was only one event that worked. 
	- You can copy the "TimeCreated SystemTime" from the "Details" section 
- Flag #1 -> `2024-05-2103:18:09`

---
2. `What is the Service Name that was targeted?`
	- In the same event just copy the "ServiceName: `MSSQLService`". That is the flag
- Flag #2 -> `MSSQLService`

---
3. `It is really important to identify the Workstation from which this activity occurred. What is the IP Address of the workstation?`
	- Again, in the same event in the "Details" section, under "EventData" there is the "IP Address"
- Flag -> `172.17.79.129`

---
4. `Now that we have identified the workstation, a triage including PowerShell logs and Prefetch files are provided to you for some deeper insights so we can understand how this activity occurred on the endpoint. What is the name of the file used to Enumerate Active directory objects and possibly find Kerberoastable accounts in the network?`
	- `Event ID 4104` in Windows Event Viewer records PowerShell Script Block Logging, capturing the content of every script block executed in PowerShell sessions. So that will be the first filter apply.
		- Leaving us with 29 events
		- In the "Details" tab we can see the path of the block of code: `C:\Users\alonzo.spire\Downloads\powerview.ps1` -> The file is the flag
- Flag #4 -> `powerview.ps1`

---
5. `When was this script executed? (UTC)`
	- Again, in the same event in the "Details" section, under "System" there is the "TimeCreated: SystemTime"
- Flag #5 -> `2024-05-21 03:16:32`

---
6. `What is the full path of the tool used to perform the actual kerberoasting attack?`
>[!hint] Parse the prefetch files using the PEcmd Tool by Eric Zimmerman. The syntax is `Pecmd.exe -d "Path of prefetchArtifacts" --csv . --csvf result.csv`. This command will create a CSV called result.csv in your current directory from where you are executing the CLI Tool. Open the CSV file in the Timeline Explorer tool(Another Eric Zimmerman tool), then look for any executables executed around the timeline we have established so far. A certain tool name will catch your eye. Then to get the path, go to the Files Loaded column and double-click the value to get a list of files interacted by the executable. It will also include its path.

- First, I had to download the `PECmd` and `TimelineExplorer` tool onto the pwnbox
	- `Pecmd.exe -d C:\Users\htb-ep-131823\Downloads\Triage\Workstation\2024-05-21T033012_triage_asset\C\Windows\prefetch --csv . --csvf result.csv`
- Next, I used `TimelineExplorer` which is a GUI tool to look at the timeline of the `CSV`
	- In `TimelineExplorer` I first look for tools that can keberoasting attacks such as:
		- `impacket`, `Rubeus`, `PowerSploit (Invoke-Kerberoast`
	- Then I filter in executables `Rubeus` which works and i get the path, this is the flag
- Flag #6 -> `C:\USERS\ALONZO.SPIRE\DOWNLOADS\RUBEUS.EXE`

---
7. `When was the tool executed to dump credentials? (UTC)`
	- Still in `TimelineExplorer` there is a column called "Last Run" which had the UTC time, this is the flag.
- Flag #7 -> `2024-05-21 03:18:08`

