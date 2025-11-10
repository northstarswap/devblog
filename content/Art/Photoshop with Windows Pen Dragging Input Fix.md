---
title: How to fix dragging small distances with a Wacom pen in Photoshop on Windows
tags:
  - art
  - software
  - fix
---
On most Windows versions while using Photoshop with a Wacom drawing tablet, there's an annoying bug where placing the pen down doesn't start drawing until you move a certain distance from the touchdown point. This is insanely annoying when trying to draw small details, requiring you to either zoom in or wiggle the pen around to begin drawing. 

Disabling Windows Ink *does* fix this issue, but it also kills your pen pressure which is one of the large points of using a drawing tablet in the first place.

These settings fix this issue without disabling Windows Ink on Windows 11 22H2 and 24H2 in my testing, but they probably work on other Windows versions as well.

First, check these settings in the modern Windows 11 settings menu:

![[res/photoshop-pen-fix/WindowsSettings.png]]

This menu can be found in the old Control Panel or by searching for it:

![[res/photoshop-pen-fix/PenAndTouch.png]]

Double click "press and hold" to access this menu:

![[res/photoshop-pen-fix/PressAndHold.png]]

Finally, make sure Windows Ink is **enabled** in your Wacom Tablet Properties:

![[res/photoshop-pen-fix/WacomSettings.png]]