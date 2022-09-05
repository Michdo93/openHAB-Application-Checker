# openHAB-Application-Checker
Use openHAB and the [Exec Action](https://www.openhab.org/docs/configuration/actions.html#exec-actions) for checking if an application is running or not. If it is running an Switch item will have the state ON. If not the state will be OFF.

## Items

You can use as example following items:

```
Switch Firefox "Firefox"
Switch VirtualBox "VirtualBox"
Switch Kodi "Kodi"
Switch VLC "VLC"
```

## Sitemaps

You can add your items as example to your sitemap like shown in the following:

```
Switch item=Firefox label="Firefox"
Switch item=VirtualBox label="VirtualBox"
Switch item=Kodi label="Kodi"
Switch item=VLC label="VLC"
```

## Rules

At first you have to add following to your `/etc/openhab/misc/exec.whitelist` file:

```
/bin/ps aux | /bin/grep [f]irefox | /usr/bin/wc -l
/bin/ps aux | /bin/grep [V]irtualBox | /usr/bin/wc -l
/bin/ps aux | /bin/grep [k]odi | /usr/bin/wc -l
/bin/ps aux | /bin/grep [v]lc | /usr/bin/wc -l
```

Off course you can run this over `SSH` and check if an application is running on a remote computer. Then as example `/bin/ps aux | /bin/grep [v]lc | /usr/bin/wc -l` will change to `/usr/bin/sshpass -p <password> /usr/bin/ssh -tt -o StrictHostKeyChecking=no <user>@<ip> "/bin/ps aux | /bin/grep [v]lc | /usr/bin/wc -l" 2> /dev/null`. Then you have to replace `<user>` and `<password>` with the username and password of your remote computer. Also you have to replace `<ip>` with the ip address of your remote computer.

```
/usr/bin/sshpass -p <password> /usr/bin/ssh -tt -o StrictHostKeyChecking=no <user>@<ip> "/bin/ps aux | /bin/grep [f]irefox | /usr/bin/wc -l" 2> /dev/null
/usr/bin/sshpass -p <password> /usr/bin/ssh -tt -o StrictHostKeyChecking=no <user>@<ip> "/bin/ps aux | /bin/grep [V]irtualBox | /usr/bin/wc -l" 2> /dev/null
/usr/bin/sshpass -p <password> /usr/bin/ssh -tt -o StrictHostKeyChecking=no <user>@<ip> "/bin/ps aux | /bin/grep [k]odi | /usr/bin/wc -l" 2> /dev/null
/usr/bin/sshpass -p <password> /usr/bin/ssh -tt -o StrictHostKeyChecking=no <user>@<ip> "/bin/ps aux | /bin/grep [v]lc | /usr/bin/wc -l" 2> /dev/null
```

With `ps aux` you can check which processes are running on a (remote) computer. With `grep` you can filter for a process. A better usage it to filter by using the application name like `vlc`. The problem is that using `grep` would also create a process. So if your application is not running there would be one process because you are using `grep` to look if there is a process or not. To fix it you can `grep` as example `[v]lc` instead of `vlc`. This means if `vlc` is not running there would be no response. If `vlc` is running there would be minimum one process because `vlc` can be running more than once at one time. At least we use `wc` for counting each line. Each process would have one line. So if `vlc` is not running with `wc` your result is `0`. If `vlc` is running with `wc` the result is the number of times `vlc` is running.

To check if an application is running or not, you either have to trigger it manually, which means you would need another `Switch` item. When its `state` changes to `ON`, a `Rule` is executed that checks if the application is running or not as just described. A better approach is to simply check periodically whether an application is running or not. This is a [time-based trigger](https://www.openhab.org/docs/configuration/rules-dsl.html#time-based-triggers). A `time-based trigger` can use `cron expression`. So we need a `cron expression` as trigger for our rule.

**However, this leads to a major problem. How often or at what periodic time interval should one check whether an application is running or not? Every second? Every minute? Every hour?**

Checking too often puts a lot of load on the computer. Checking too infrequently, on the other hand, leads to an inaccurate result. Possibly then to an erroneous one, because it was still assumed that an application is running and in the meantime it has already been terminated. Or just the other way around: The application is running in the meantime, but it is assumed that it is not running.

So the following rule is just an example of how this concept might work:

```
rule "Application checker"
when
    Time cron "0/1 * * ? * * *"
then
    var firefox = executeCommandLine("/bin/ps", "aux", "|", "/bin/grep", "[f]irefox", "|", "/usr/bin/wc", "-l")
    logInfo("application check firefox", firefox)
    if (firefox == "0") {
        Firefox.postUpdate(OFF)
    } else {
        Firefox.postUpdate(ON)
    }
    
    var virtualbox = executeCommandLine("/bin/ps", "aux", "|", "/bin/grep", "[V]irtualBox", "|", "/usr/bin/wc", "-l")
    logInfo("application check virtualbox", virtualbox)
    if (virtualbox == "0") {
        VirtualBox.postUpdate(OFF)
    } else {
        VirtualBox.postUpdate(ON)
    }
    
    var kodi = executeCommandLine("/bin/ps","aux","|","/bin/grep","[k]odi","|","/usr/bin/wc","-l")
    logInfo("application check kodi", kodi)
    if (kodi == "0") {
        Kodi.postUpdate(OFF)
    } else {
        Kodi.postUpdate(ON)
    }
    
    var vlc = executeCommandLine("/bin/ps","aux","|","/bin/grep","[v]lc","|","/usr/bin/wc","-l")
    logInfo("application check vlc", vlc)
    if (vlc == "0") {
        VLC.postUpdate(OFF)
    } else {
        VLC.postUpdate(ON)
    }
end
```

If you want to use it over `ssh` it looks like following:

```
rule "Application checker"
when
    Time cron "0/1 * * ? * * *"
then
    var firefox = executeCommandLine(Duration.ofSeconds(1), "/usr/bin/sshpass","-p","<password>","/usr/bin/ssh","-tt","-o","StrictHostKeyChecking=no","<user>@<ip>","ps","aux","|","/bin/grep","[f]irefox","|","/usr/bin/wc","-l","2>","/dev/null")
    firefox = firefox.replace("Connection to <ip> closed.", "")
    firefox = firefox.replace("\r\n", "")
    logInfo("application check firefox", firefox)
    if (firefox == "0") {
        Firefox.postUpdate(OFF)
    } else {
        Firefox.postUpdate(ON)
    }

    var virtualbox = executeCommandLine(Duration.ofSeconds(1), "/usr/bin/sshpass","-p","<password>","/usr/bin/ssh","-tt","-o","StrictHostKeyChecking=no","<user>@<ip>","ps","aux","|","/bin/grep","[V]irtualBox","|","/usr/bin/wc","-l","2>","/dev/null")
    virtualbox = virtualbox.replace("Connection to <ip> closed.", "")
    virtualbox = virtualbox.replace("\r\n", "")
    logInfo("application check virtualbox", virtualbox)
    if (virtualbox == "0") {
        VirtualBox.postUpdate(OFF)
    } else {
        VirtualBox.postUpdate(ON)
    }
    
    var kodi = executeCommandLine(Duration.ofSeconds(1), "/usr/bin/sshpass","-p","<password>","/usr/bin/ssh","-tt","-o","StrictHostKeyChecking=no","<user>@<ip>","ps","aux","|","/bin/grep","[k]odi","|","/usr/bin/wc","-l","2>","/dev/null")
    kodi = firefox.replace("Connection to <ip> closed.", "")
    kodi = firefox.replace("\r\n", "")
    logInfo("application check firefox", firefox)
    if (kodi == "0") {
        kodi.postUpdate(OFF)
    } else {
        kodi.postUpdate(ON)
    }

    var vlc = executeCommandLine(Duration.ofSeconds(1), "/usr/bin/sshpass","-p","<password>","/usr/bin/ssh","-tt","-o","StrictHostKeyChecking=no","<user>@<ip>","ps","aux","|","/bin/grep","[v]lc","|","/usr/bin/wc","-l","2>","/dev/null")
    vlc = vlc.replace("Connection to <ip> closed.", "")
    vlc = vlc.replace("\r\n", "")
    logInfo("application check vlc", vlc)
    if (vlc == "0") {
        VLC.postUpdate(OFF)
    } else {
        VLC.postUpdate(ON)
    }
end
```

And of course you can add other rules for running or stopping an application like following:

```
rule "Run VLC"
when
    Item VLC received command ON
then
    executeCommandLine("DISPLAY=:0","/usr/bin/vlc","--fullscreen")
end

rule "Quit VLC"
when
    Item VLC received command OFF
then
    executeCommandLine("/usr/bin/sudo","killall","/usr/bin/vlc")
end
```


Or if you want to do it over `ssh` it looks like following:

```
rule "Run VLC"
when
    Item VLC received command ON
then
    executeCommandLine("/usr/bin/sshpass","-p","<password>","/usr/bin/ssh","-t","-o","StrictHostKeyChecking=no","<user>@<ip>","DISPLAY=:0","/usr/bin/vlc","--fullscreen")
end

rule "Quit VLC"
when
    Item VLC received command OFF
then
    executeCommandLine("/usr/bin/sshpass","-p","<password>","/usr/bin/ssh","-t","-o","StrictHostKeyChecking=no","<user>@<ip>","/usr/bin/sudo","killall","/usr/bin/vlc")
end
```
