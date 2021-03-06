QTestService
============

QTestService - Service for running a process on the Winlogon desktop

This is my fork of omeg's QTestService from his article [Running processes on the Winlogon desktop](http://omeg.pl/blog/2014/01/running-processes-on-the-winlogon-desktop/).

Please heed to the author's warning: ***[Running a process on the Winlogon desktop] is a Bad Idea unless you know exactly what you�re doing, why you�re doing it and there are no alternatives.***


Know This
---------

### Your process runs in the security context of user SYSTEM.

Do not forget this.

### There is a delay before your process is created.

On the Windows 7 x64 SP1 computer that I am running QTestService on it can take up to 10 seconds before the service receives notification that the console has been connected to a different session. I don't know what causes the delay. In the Windows Application event log I see this info message from Winlogon:

`The winlogon notification subscriber <SessionEnv> was unavailable to handle a notification event.`

QTestService only starts your process after it has received the notification.

### Error: CreateProcessAsUser failed, GetLastError: 5

If your process is being created in a session that has already ended you will see this message in the log. It is a likely message if you are quickly switching between two already logged in users because in that case the switch user session only exists during the switch. As noted above there can be a slight delay between what actually is happening and the notification messages received, so unless you see this error all the time assume it failed because the session ended before QTestService received notification.

### There are multiple Winlogon desktops.

Although we may refer to a singular Winlogon desktop there is actually a Winlogon desktop for each session. Therefore if multiple sessions exist there are multiple Winlogon desktops. This service only executes your process in the session that you next connect to after the service is started (unless you have defined `LOOP_WORKER` -- see below).

### A process can be executed multiple times for a session.

If QTestService is built with `LOOP_WORKER` defined then for each console connection a process is executed.

For example:
- You start QTestService
- You, a logged in user in session 1, switch to a different logged in user in session 4.
- Your process is executed on the Winlogon desktop in session 4.
- You, in session 4, switch back to the user in session 1.
- Your process is executed on the Winlogon desktop in session 1.
- You, in session 1, switch back to the user in session 4.
- Your process is executed on the Winlogon desktop in session 4.

The example does not account for a process that may be created for each temporary switch user session, assuming you do not switch fast (refer to the process delay issue).

### The Winlogon desktop has a ****very**** small desktop heap.

All user objects (read: GUI stuff) consume memory from the heap of the desktop they are on. Be careful you don't run a process that could exhaust a Winlogon desktop heap. Its heap is 20-150 times smaller than an interactive desktop heap.

### Your process may appear corrupted.

Separate from the desktop heap issue above there is an issue where a graphics settings change may cause your process' GUI to no longer display properly. I observed this on the Windows 7 x64 SP1 computer that I am running QTestService on. Here is how it happens:

- You start QTestService
- You, a logged in user in some session, switch to the switch user screen.
- You wait at the switch user screen for your process to be executed.
- Your process is executed in the temporary session of the switch user screen.
- You then login as a user that isn't already logged in.

What happens next is the temporary session for the switch user screen is no longer temporary and is assigned to the newly logged in user as their session. When I did this and the cleartype and graphics settings were different, the process opened before the login (I tried cmd.exe and Process Hacker) did not adjust to the new settings and its controls became unresponsive, blacked out or inverted.

### Reference

- [Sessions, Desktops and Windows Stations](http://blogs.technet.com/b/askperf/archive/2007/07/24/sessions-desktops-and-windows-stations.aspx)
- [Desktop Heap Overview](http://blogs.msdn.com/b/ntdebugging/archive/2007/01/04/desktop-heap-overview.aspx)
- [Desktop Heap Monitor for Windows Vista, Server 2008 and 7](http://blog.airesoft.co.uk/2009/10/desktop-heap-monitor-vista-7/)
- [Pushing the Limits of Windows: USER and GDI Objects](http://blogs.technet.com/b/markrussinovich/archive/2010/02/24/3315174.aspx)
- [Running processes on the Winlogon desktop](http://omeg.pl/blog/2014/01/running-processes-on-the-winlogon-desktop/)
- [Displaying a program on the Windows secure desktop](http://calebdelnay.com/blog/2012/01/displaying-a-program-on-the-windows-secure-desktop)
