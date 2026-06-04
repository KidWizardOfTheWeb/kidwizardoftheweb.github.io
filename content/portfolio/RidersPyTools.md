+++
date = '2026-06-04T16:29:07-04:00'
draft = false
title = 'RidersPyTools'
author = 'KC'
authorTwitter = "" #do not include @
cover = ""
tags = ["Python", "Dolphin", "GameCube", "Games"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
+++


## RidersPyTools: Python framework for memory address bindings

#### May 2026-Present

Github repository: https://github.com/KidWizardOfTheWeb/RidersPyTools

Work in progress.
Features to be added:
- More memory addresses from the original Sonic Riders
- Symbol map parsing full implementation.
- Dynamic class creation, possibly.

RidersPyTools allows developers to replicate memory structures from their original games and create code that can hook into a live emulated instance and modify the game in real time without having to use constant Dolphin Memory Engine calls. Everything is done implicitly, while code writing can be formatted as simply or as accurate as the game being used.

This package was originally built with Sonic Riders in mind, but is flexible enough to insert any structures specific to a game implementation.

For Sonic Riders, the structure implementation matches the disassembly and the decompiled code that is already known, being almost analogous to SRTE's C++ codebase. This serves as an easy bridge to learn how to code for riders in particular by starting here and transferring experience over, as syntax is 99% similar between our python implementation and C++ implementation.

**Tech stack**:
- Python
- Dolphin Memory Engine
