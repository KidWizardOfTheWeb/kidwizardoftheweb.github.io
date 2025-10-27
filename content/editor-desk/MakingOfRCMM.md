+++
date = '2025-10-26T20:30:19-04:00'
title = 'Solving file maintenance: how RCMM was developed.'
author = "KC"
authorTwitter = "" #do not include @
cover = ""
#tags = ["", ""]
keywords = ["", ""]
description = ""
showFullContent = true
readingTime = false
hideComments = false
draft = false
+++


## R.C.M.M. - A file management solution for GameCube Mods

At the time of writing (when I started this website), this is my most recent venture (I'm pretty proud of it quite honestly).

My aim was to fix a common issue with developing mods involving file changes to emulated GameCube and Wii games, which is build and file management.

While I was developing Sonic Riders Tournament Edition, most of the time players were generally discouraged from modifying the game in any way due to being unable to play together with someone if they do not have the *exact* same setup as you. That meant **no custom models or music packs** unless you gave the other player an Xdelta (file used to modify a base file into the target output) and have them patch their game copy. 

This was a hassle, as the mod creator always had to update the **Xdelta** patcher every time a new patch for Tournament Edition was released to stay up to date.