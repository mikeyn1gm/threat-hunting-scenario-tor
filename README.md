<img width="400" src="https://github.com/user-attachments/assets/44bac428-01bb-4fe9-9d85-96cba7698bee" alt="Tor Logo with the onion and a crosshair on it"/>

# Threat Hunt Report: Unauthorized TOR Usage
- [Scenario Creation](https://github.com/mikeyn1gm/threat-hunting-scenario-tor/blob/main/threat-hunting-scenario-tor-event-creation.md)

## Platforms and Languages Leveraged
- Windows 10 Virtual Machines (Microsoft Azure)
- EDR Platform: Microsoft Defender for Endpoint
- Kusto Query Language (KQL)
- Tor Browser

##  Scenario

Management suspects that some employees may be using TOR browsers to bypass network security controls because recent network logs show unusual encrypted traffic patterns and connections to known TOR entry nodes. Additionally, there have been anonymous reports of employees discussing ways to access restricted sites during work hours. The goal is to detect any TOR usage and analyze related security incidents to mitigate potential risks. If any use of TOR is found, notify management.

### High-Level TOR-Related IoC Discovery Plan

- **Check `DeviceFileEvents`** for any `tor(.exe)` or `firefox(.exe)` file events.
- **Check `DeviceProcessEvents`** for any signs of installation or usage.
- **Check `DeviceNetworkEvents`** for any signs of outgoing connections over known TOR ports.

---

## Steps Taken

### 1. Searched the `DeviceFileEvents` Table

Searched for any file that had the string "tor" in it and discovered what looks like the user "mikeylab" downloaded a TOR installer, did something that resulted in many TOR-related files being copied to the desktop, and the creation of a file called `tor-shopping-list.txt` on the desktop at `2025-02-27T06:41:56.0186683Z`. These events began at `2025-02-27T06:18:48.8183897Z`.

**Query used to locate events:**

```kql
DeviceFileEvents  
| where DeviceName == "mikey-win10-vla"  
| where InitiatingProcessAccountName == "mikeylab"  
| where FileName contains "tor"  
| where Timestamp >= datetime(2025-02-27T06:18:48.8183897Z)  
| order by Timestamp desc  
| project Timestamp, DeviceName, ActionType, FileName, FolderPath, SHA256, Account = InitiatingProcessAccountName
```
<img width="1212" alt="image" src="https://github.com/user-attachments/assets/52c9c08f-fd5e-4da2-bd03-4628f5f1549e">

---

### 2. Searched the `DeviceProcessEvents` Table

Searched for any `ProcessCommandLine` that contained the string "tor-browser-windows-x86_64-portable-14.0.6.exe". Based on the logs returned, at `2025-02-27T06:22:42.9937103Z`, an employee on the "mikey-win10-vla" device ran the file `tor-browser-windows-x86_64-portable-14.0.6.exe` from their Downloads folder, using a command that triggered a silent installation.

**Query used to locate event:**

```kql

DeviceProcessEvents  
| where DeviceName == "mikey-win10-vla"  
| where ProcessCommandLine contains "tor-browser-windows-x86_64-portable-14.0.6.exe"  
| project Timestamp, DeviceName, AccountName, ActionType, FileName, FolderPath, SHA256, ProcessCommandLine
```
<img width="1212" alt="image" src="https://github.com/user-attachments/assets/7470eae8-19b4-412f-8fd3-f92606fb54b0">


---

### 3. Searched the `DeviceProcessEvents` Table for TOR Browser Execution

Searched for any indication that user "mikeylab" actually opened the TOR browser. There was evidence that they did open it at `2025-02-27T06:23:55.1381116Z`. There were several other instances of `firefox.exe` (TOR) as well as `tor.exe` spawned afterwards.

**Query used to locate events:**

```kql
DeviceProcessEvents  
| where DeviceName == "mikey-win10-vla"  
| where FileName has_any ("tor.exe", "firefox.exe", "tor-browser.exe")  
| project Timestamp, DeviceName, AccountName, ActionType, FileName, FolderPath, ProcessCommandLine, SHA256  
| order by Timestamp desc
```
<img width="1212" alt="image" src="https://github.com/user-attachments/assets/fa409971-36ff-4da9-995f-77d06cecb984">


---

### 4. Searched the `DeviceNetworkEvents` Table for TOR Network Connections

Searched for any indication the TOR browser was used to establish a connection using any of the known TOR ports. At `2025-02-27T06:24:20.5898172Z`, an employee on the "mikey-win10-vla" device successfully established a connection to the local IP address `127.0.0.1` on port `9150`. The connection was initiated by the process `firefox.exe`, located in the folder `c:\users\mikeylab\desktop\tor browser\browser\torbrowser\tor\tor.exe`. There were a couple of other connections to sites over port `443`.

**Query used to locate events:**

```kql
DeviceNetworkEvents  
| where DeviceName == "mikey-win10-vla"  
| where InitiatingProcessAccountName != "system"  
| where InitiatingProcessFileName in ("tor.exe", "firefox.exe")  
| where RemotePort in ("9001", "9030", "9040", "9050", "9051", "9150", "80", "443")  
| project Timestamp, DeviceName, InitiatingProcessAccountName, ActionType, RemoteIP, RemotePort, InitiatingProcessFileName, InitiatingProcessFolderPath  
| order by Timestamp desc
```
<img width="1212" alt="image" src="https://github.com/user-attachments/assets/c4d0acc5-8430-43cd-adcb-dca03bb36acd">


---

## Chronological Event Timeline 

### 1. File Download - TOR Installer

- **Timestamp:** `2025-02-27T06:18:48.8183897Z`
- **Event:** The user "mikeylab" downloaded a file named `tor-browser-windows-x86_64-portable-14.0.6.exe` to the Downloads folder.
- **Action:** File download detected.
- **File Path:** `C:\Users\mikeylab\Downloads\tor-browser-windows-x86_64-portable-14.0.6.exe`

### 2. Process Execution - TOR Browser Installation

- **Timestamp:** `2025-02-27T06:22:42.9937103Z`
- **Event:** The user "mikeylab" executed the file `tor-browser-windows-x86_64-portable-14.0.6.exe` in silent mode, initiating a background installation of the TOR Browser.
- **Action:** Process creation detected.
- **Command:** `tor-browser-windows-x86_64-portable-14.0.6.exe /S`
- **File Path:** `C:\Users\mikeylab\Downloads\tor-browser-windows-x86_64-portable-14.0.6.exe`

### 3. Process Execution - TOR Browser Launch

- **Timestamp:** `2025-02-27T06:23:55.1381116Z`
- **Event:** User "mikeylab" opened the TOR browser. Subsequent processes associated with TOR browser, such as `firefox.exe` and `tor.exe`, were also created, indicating that the browser launched successfully.
- **Action:** Process creation of TOR browser-related executables detected.
- **File Path:** `C:\Users\mikeylab\Desktop\Tor Browser\Browser\firefox.exe`

### 4. Network Connection - TOR Proxy Usage

- **Timestamp:** `2025-02-27T06:24:20.5898172Z`
- **Event:** The Tor Browser (firefox.exe) established a successful connection to `127.0.0.1` on port `9150` by user “mikeylab”, indicating the activation of the Tor proxy.
- **Action:** Connection detected.
- **Remote IP:** 127.0.0.1
- **Remote Port:** 9150

### 5. Network Connection - Web Traffic Over Tor

- **Timestamp:** `2025-02-27T06:24:20.5898172Z`
- **Event:** The Tor Browser (tor.exe) established connections to external sites over port `443`, indicating possible internet browsing through the Tor network.
- **Action:** Connection detected.
- **Remote IP:** 104.152.111.1
- **Remote Port:** 443

### 6. File Creation - TOR Shopping List

- **Timestamp:** `2025-02-27T06:41:56.0186683Z`
- **Event:** The user "mikeylab" created a file named `tor-shopping-list.txt` on the desktop, potentially indicating a list or notes related to their TOR browser activities.
- **Action:** File creation detected.
- **File Path:** `C:\Users\mikeylab\Desktop\tor-shopping-list.txt`

---

## Summary

The user "mikeylab" on the "mikey-win10-vla" device initiated and completed the installation of the TOR browser. They proceeded to launch the browser, establish connections within the TOR network, and created various files related to TOR on their desktop, including a file named `tor-shopping-list.txt`. This sequence of activities indicates that the user actively installed, configured, and used the TOR browser, likely for anonymous browsing purposes, with possible documentation in the form of the "shopping list" file.

---

## Response Taken

TOR usage was confirmed on the endpoint `mikey-win10-vla` by the user `mikeylab`. The device was isolated, and the user's direct manager was notified.

---
