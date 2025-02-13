---
author: Muhammad Najmi bin Ahmad Zabidi 
title: 'Getting output from JPS with NRPE'
tags: 
- linux
- monitoring
- nrpe
- nagios
- icinga
- java 
gh_issue_number: 1655
---

### Nagios and NRPE
The hosting team’s routine typically involves the server’s and site monitoring task. One of the tools that is being used for monitoring is Icinga (which is based on Nagios). When monitoring the host resources, one of the tools that we could use is Nagios Remote Plugin Executor or NRPE. 

We encountered an issue when executing NRPE, the Nagios agent that runs on servers being monitored didn’t give the expected output which is similar compared to when the script was executed on the server itself. Usually the NRPE-related call should not be an issue to be executed on the target server as it will be declared in the sudoers (commonly /etc/sudoers) file. In this writing, I will explain the situation when I encountered an issue to get the output from jps (Java Virtual Machine Process Status Tool), which only could be executed as the “root” user on the terminal. 

### Examples
To get the process’ state (in this case, the process is the “Hello World” process) from Icinga’s head server. 

Let’s say we have a small program with the name of Hello.java

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
        
        try {
            Thread.sleep(3000);  // Sleep for 3 seconds
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```


Run the program after we compile it
```plain
java Hello.java 
Hello, World!
``` 
(the program will stick around on the terminal until the time out)

Checking the process with `ps`
```plain
# ps aux|grep Hello|grep -v grep
najmi      84335  1.3  0.2 11535904 88128 pts/6  Sl+  12:15   0:00 java Hello.java
```
Checking with jps will show the process ID
```plain
# sudo -s -u najmi jps -l
84442 jdk.jcmd/sun.tools.jps.Jps
84335 jdk.compiler/com.sun.tools.javac.launcher.Main
```
But the nrpe user does not see it:
```plain
# sudo -s -u nrpe  jps -l
84643 jdk.jcmd/sun.tools.jps.Jps
```

So instead of running the `jps` command directly as Nagios, we let the system run (as root) to run jps and dump the result onto a file. NRPE-based script later will read the output and feed the result to the dashboard. 

Consider the following example of a bash script, which will dump the Java process ID inside a temporary file. We can use this script to be invoked as an NRPE script.
```bash
#!/bin/bash

# Check if program name was provided
if [ -z "$1" ]; then
    echo "UNKNOWN: Please specify a program name to check"
    exit 3
fi

TEMP_FILE="/tmp/java_pid.txt"

# Clear the temp file first
> $TEMP_FILE

# Run jps and save output to temp file
jps -l > $TEMP_FILE

# Check if program is in the temp file
if grep -q "$1" $TEMP_FILE; then
    PID=$(grep "$1" $TEMP_FILE | awk '{print $1}')
    echo "OK - $1 is running with PID $PID"
    exit 0
else
    echo "CRITICAL - $1 is down"
    exit 2
fi
```
The given example could be adapted with your use case.

### Conclusion

This write up is just an example when we need to get the output from JPS (say, for whatever reason that we could not use the normal “ps” or “pgrep” command).

There could be another way to solve this issue (which I might not be aware of). The method which I shared above is just one of the ways to get jps’ report works with Icinga/Nagios plugin. Please let me know if you have experience with jps and Icinga/Nagios - and how do you handle the reporting. 



Related readings:

https://opensource.com/article/21/10/check-java-jps

https://docs.oracle.com/javase/7/docs/technotes/tools/share/jps.html

