---
layout: default
title: How to ignore laptop lid close
---
# How to ignore laptop lid close in Fedora 23

Since I often use my laptop as a desktop PC, I want it to keep running when I close its lid. Unfortunately, it goes into Suspend mode, eventually severing all network connections.

Gnome has the Tweak Tool, a GUI tool to change a number of settings including the system's reaction to closing the lid, but this particular setting doesn't seem to be persistent. When I restart the laptop in the morning, it has forgotten that I didn't want it to suspend itself.

It turns out that systemd manages this particular detail. It can be configured in `/etc/systemd/logind.conf:

```
$ sudo vi /etc/systemd/logind.conf
HandleLidSwitch=ignore
$ sudo systemctl restart systemd-logind
```
The setting is case-sensitive; a value of `Ignore` doesn't work:
```
$ sudo journalctl -u systemd-logind
G
systemd-logind[4372]: [/etc/systemd/logind.conf:24] Failed to parse handle action setting, ignoring: Ignore
...
systemd-logind[4372]: Lid closed.
systemd-logind[4372]: Suspending...
systemd-logind[4372]: Lid opened.
```

Reference: https://ask.fedoraproject.org/en/question/27808/preventing-lid-close-suspension/

