+++
date = '2025-10-26T20:30:19-04:00'
title = 'Solving file maintenance: how RCMM was developed.'
author = "KC"
authorTwitter = "" #do not include @
cover = ""
tags = ["Python", "GameCube", "Tools"]
keywords = ["", ""]
description = ""
showFullContent = true
readingTime = false
hideComments = false
draft = false
+++

{{< toc >}}

## R.C.M.M. - A file management solution for GameCube Mods

At the time of writing (when I started this website), this is my most recent venture (I'm pretty proud of it quite honestly).

My aim was to fix a common issue with developing mods involving file changes to emulated GameCube and Wii games, which is build and file management.

### The story
While I was developing Sonic Riders Tournament Edition (SRTE), most of the time players were generally discouraged from modifying the game in any way due to being unable to play together with someone if they do not have the *exact* same setup as you. That meant **no custom models or music packs** unless you gave the other player an Xdelta (file used to modify a base file into the target output) and have them patch their game copy. 

Xdelta is a file that contains all diffs between a base and a target file. This could take anywhere from 15 minutes to an hour to compile, since you cannot multithread the bitchecks it has to run on everything. If even one byte is out of place, the whole program could be out of commision.

This was a hassle, as the mod creator always had to update the **Xdelta** patcher every time a new patch for Tournament Edition was released to stay up to date. In fact, mods like [Akaneia](https://github.com/akaneia/akaneia-build/releases) for Super Smash Bros. Melee still use Xdelta to this day to patch their game. 

### The Xdelta problem & trying for longterm growth
For longterm releases with smaller/more inconsistent playerbases, this is fine. But for longterm growth on the other hand, Xdeltas only hamper the entire onboarding process. Not only do you have to make it work for those who cannot operate their filesystem on a regular basis, but even something as simple as "drag-and-drop" cannot be taken for granted as a skill anymore (I've learned this the hard way while developing tools for general usage).

Longterm growth was my goal, so fiddling with Xdeltas was out of the question. With SRTE, I ended up developing a patcher that automatically grabs updates from a server we host the delta files on (shoutout to [Sewer56](https://github.com/sewer56) for their implementation of delta patch files) and patches the files locally after retrieval. All the user had to do was make sure they had .NET 7 and place the ISO in the proper folder. 

Even then, the workflow still looked like this to make a new patch and retrieve it:
1. Compile the extracted files into an ISO (<1 min)
2. Use Sewer's delta generator to generate base ISO/new ISO hashes and patch files (15-30 minutes)
3. Test ISO (5-30 minutes, depending on changes added).
4. Upload to server (30 minutes).
5. Test server retrieval with patcher (<1-2 minutes on my computer & network).

That's about an hour to create new files, test, release, and test the release each time. And if you screw up something or leave something in? Gotta redo the whole process. Every time I upload a new patch for SRTE, I am anxious 100% of the way through for that reason. It's happened before, and it'll happen again.

### Studying other solutions: [HedgeModManager](https://github.com/thesupersonic16/HedgeModManager)
Last year, I played Shadow Generations and got to use a little program called HedgeModManager for the first time. I've never really done PC game modding before, and I marveled at how simple it was to operate and understand. The connectivity with GameBanana blew my mind at the time as well. I just couldn't fathom how the connection from website to launcher worked so well and effortlessly. 

So when I thought of making a mod manager, HedgeModManager became my idea of a perfect manager. Their repository was easy enough to comprehend what the general idea for implementing their systems was, but there was a general difference between the types of games handled in my situation versus PC.

See, PC games use clients like Steam or Epic to handle versioning, file management, and storage. With Sonic PC games, file and code injection was the main methodology to make mods run, using .dlls to do the injections.

With games you run on Dolphin (GameCube/Wii), the story is a bit different. You can extract the files from any given ISO and still play said extracted files as a full ISO. Replacing those files (with valid ones ofc) also works, as Dolphin will load whatever is requested by the game and if it follows the file path/file name convention. So for my version of a mod manager, the basis had to be **layering** instead of injection.

Knowing this, I sketched out a few possible ideas on how this could work. Python is always on my mind for simple solutions quick (it really is a "do-everything" kind of language, I love it), and when I revisted some of my older projects for inspiration, dictionaries (hashmaps) stuck out to me as a solution. Python's specific collision handling implementation had a certain behavior that mirrored my idea of layering perfectly. So, I got to work immediately. 

### The conceptual workflow of R.C.M.M.

R.C.M.M.'s workflow goes like this:
1. Add a game to the mod manager. This uses DolphinTool as a subprocess to extract the ISO to a new directory in a mods folder, named after the game ID.
2. The extracted ISO is then used as a "base case" for laying mods on top of. Think of it like a smoothie bowl, where you need a base to put your toppings on top of, otherwise the toppings do nothing on their own.
3. Any mods the user turns on will then calculate a database file for all the files in said mod. This database is actually a dictionary (hashmap) of all the files, which has an important paradigm in the python implementation that we'll use later. The extracted ISO also has it's own database.
4. Once all mods are turned on and the user is ready to save, when pressing the save button, all database files are combined. The way this works revolves around hashmap collision handling. Python has a really neat way of handling when it detects two of the same keys between dictionaries: 

Let's say dictionary A has key-pair `{bigKeyGuy: fooValue}` and dictionary B has key-pair `{bigKeyGuy: barValue}`. If the operation `dictionary A + dictionary B` happens, python will detect two `bigKeyGuy` keys exist and *replaces the value of the dictionary being added to*.

E.g.
```py

# Generate our dictionaries
dictA = {"bigKeyGuy": "fooValue"}

dictB = {"bigKeyGuy": "barValue"}

# Print our dictionary A to show off value
print(dictA) -> {"bigKeyGuy": "fooValue"}

# Add dictionaries together, python implicitly handling our hashmap collision during this
dictA = dictA.update(dictB)

# Print our dictionary A to show off newvalue
print(dictA) -> {"bigKeyGuy": "barValue"}
```

So `dictA`'s key's value was replaced by `dictB`'s key's value while keeping the same key. No errors! Unlike some other languages.

Using this behavior, I designed a database schema for the base ISO, the modded output ISO, and each mod.

Essentially, all we have to do is record:
- The subdirectories of the mod ("sys"/"files" for GameCube).
- What files exist in the ISO/mod.
- Where the original directory was.
- The hash of the file (for file integrity purposes).

Example of the schema using a mod that includes a sys folder with a .dol and a single file replacement in the files folder:
```py
{
    "sys": {
        "main.dol": [
            "C:\\Users\\XXX\\Downloads\\RidersDolphin3Windows\\x64\\ReaperCoMods\\GXEE8P\\GXEE8P_mods\\SRTE v2.4.6\\sys\\main.dol",
            ".dol",
            "65a9b1be9c52e53900e073cd1be5123941581a31816ee107609a78d77c69796f"
        ]
    },
    "files": {
        "0": [
            "C:\\Users\\XXX\\Downloads\\RidersDolphin3Windows\\x64\\ReaperCoMods\\GXEE8P\\GXEE8P_mods\\SRTE v2.4.6\\files\\0",
            "",
            "ff1385b571519f676af815a71718944c6793d4fc3e726eb89b4eaf6e3124b7ae"
            ]
        }
}
```

Remember how I said the extracted ISO was our base case? Using the above logic, this is how we handle adding mods on top to make our final modded ISO:

```py
# This is simplified from the real version for readability
# Real version: 
# https://github.com/KidWizardOfTheWeb/ReaperCo.ModManager/blob/master/modfileutils.py#L241

# Get ORIGINAL ISO database first
original_iso_db = ORIGINAL_ISO_DIR

# Make the dict we'll pass for final merge
combined_file_dict = {}

# Load the original ISO first into the dictionary
with open(os.path.join(original_iso_db, "db.json"), "r") as file:
    combined_file_dict = json.load(file)

# Load all other dictionaries, OR (the operator, using update) them into the combined one
# Assume active_mods is a list
for index in range(len(active_mods)):

    # Assume mod_found_path is a list of paths for each active mod
    # Load the databases for each active mod
    with open(os.path.join(Path(mod_found_path[index]), "db.json"), "r") as file:
        
        # Load file into this variable
        new_dict_to_add = json.load(file)
        
        # Get the keys of the newly loaded dictionary, update that key in combined_file_dict to lay on top of our base ISO
        new_dict_keys = new_dict_to_add.keys()
        
        # The "keys" here are the subdirectories in the mods. Namely, "sys" and "files" for gamecube games.
        # Python only updates the files that were added in the mod and doesn't replace any that aren't in the subdirectories
        for keys in new_dict_keys:
            combined_file_dict[keys].update(new_dict_to_add[keys])

# Write dict database to gameID_MOD folder for final file merging
mod_iso_db = MOD_ISO_DIR

# Dump JSON to modded ISO output, so all merged files can be retrieved and copied to here in the next step.
with open(os.path.join(mod_iso_db, Path("db.json")), "w") as file:
    json.dump(combined_file_dict, file, indent=4)

return mod_iso_db
```

Once the database is saved, how is the actual modified ISO created?

Well since we made a hashmap for the final output, quite easily! Check it:

```py

# This is the full version actually, but link anyway:
# https://github.com/KidWizardOfTheWeb/ReaperCo.ModManager/blob/master/modfileutils.py#L296

def move_mod_files_to_final_place(mod_iso_db):
    # Get file dict from... file
    combined_file_dict = {}
    with open(os.path.join(mod_iso_db, Path("db.json")), "r") as file:
        combined_file_dict = json.load(file)

    # Go through every key (top-level directory), then every key-value[0] to get the file directory to move into gameID_MOD

    # Get top level directory, then filelist
    for directory, filelist in combined_file_dict.items():
        # Then get filename, then directory (filedata[0])
        # Use this to move.

        # Make the directory first before each iteration
        new_directory = os.path.join(Path(mod_iso_db), Path(directory))

        # Clear this directory first to avoid holdover files
        if os.path.exists(new_directory):
            shutil.rmtree(new_directory)

        # Generate the directory again and place files
        os.makedirs(new_directory, exist_ok=True)

        for filename, filedata in filelist.items():
            shutil.copy(filedata[0], new_directory)
            pass
        pass
    pass
```

With this final loop, we essentially retrieve the directories of all files on the modded ISO hashmap and copy it to the final ISO's directory. 

Now when dolphin loads the modded ISO directory, all of the modded files are already included and ready to be played. Best part is, if other users turn on the same mods as you, you can play your mods online with each other!

### Initial release and conclusion

So far with the first release (along with bugfixes) of R.C.M.M., the SRTE community has been able to use their music and model mods together online. You can check out some creations in the [Sonic Riders Competitive](https://discord.com/invite/sonicriders) Showcase channel. More features are planned for R.C.M.M. in the future to make implementation even easier for users (and hopefully a proper binary for full release). There might be some caveats like sending files manually for now, but the modpack specs mean that it's as easy as adding a folder and dropping in and refreshing the window.

I'm currently asking around for other GameCube games and seeing where R.C.M.M. can do some good. If you have a game you'd like to try, give [R.C.M.M.](https://github.com/KidWizardOfTheWeb/ReaperCo.ModManager/tree/master) a shot!

This is also the first project I'm releasing under the Reaper Co. moniker. I hope to release more over time and maybe even garner a team. But that's for the future.

Thank you for reading my first post! You can message me on discord at `kid.chameleon` or leave an issue on my GitHub for new changes!