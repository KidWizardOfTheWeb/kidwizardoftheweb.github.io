+++
date = '2026-01-12T17:45:21-05:00'
draft = false
title = 'January Project Update'
+++

I missed a December update due to a lot going on at once and still getting used to posting things. It's not a common thing for me to post sometimes, as I keep my head down and work more often than not.

Anyways, here's:

## What's happened this January?

### The job search: still looking, but hopeful.

Still looking out there for anything to open up, but there are some prospects possibly. Here's hoping to landing one this year! Hopefully soon, I got a lot to offer and no position to occupy to exhibit that.

### R.C.M.M. fixes: a LOT of refactoring later...

On first release, I was *extremely* confident in the mod manager handling all GameCube games. Until I saw a TON used subfolders and it did not agree with games like that at all due to not expecting those. Sonic Riders had zero subfolders, so my expectation was set for that.

So in the last few days, I took some time to study the filesystems of games like Shadow the Hedgehog (2005, GC) to modify it to work properly in those contexts.

I'd say a big hurdle came from the biggest boon of the architected system actually, that being the `db.json` files constructed. These files essentially are the directory maps to orchestrate the entirety of the file management process.

Here's a quick recap on how those worked:

1. R.C.M.M. generates a database (`db.json`) file for every mod and game. It usually regenerates these when files are saved with the save button.
2. These database files contain a hashmap representation of the directory of the mod/game. All directories are key-dict pairs, while all files are key-list pairs. We treat the dicts as directories, while lists contain the file name as a key and related file data as a list of data.
3. During the final save process, R.C.M.M. goes through all activated mods and reads their db.json file, using the directories as keys and replacing the values of the directories from the original filesystem (from the base game). Once this is done, the final output hashmap is produced, containing all of the paths to the modded files layered on top of the original filesystem.
4. The mod manager then goes through the final output hashmap and copies the paths of all files in the hashmap to the MOD folder. These paths are either from the original game's filesystem (for unmodified files) or from the mods we've enabled (as we replaced the paths of the hashmap from the original game with our modified files).
5. You can now play this extracted version of the game ISO or compile it to a full ISO.

The problem was that since subdirectories weren't accounted for originally, the mod manager ignored those entirely and never copied over files in subdirectories. Thus, I extended the schema for the db.json files to make key-dict entries possible and recognized as directories. This already worked for the `sys/files` directories earlier, so I just had to search for subdirectories once all files were handled on the current directory level.

Doing this for database creation was a bit tedious at first, as in python, you can't exactly "go back up a level" in a dictionary. But using some concepts I googled around for and some techniques I learned from hackerrank from practice problems, I used recursion to fix the problem.

See by using recursion, once you call return, you technically end up going back a level already, so when we build the db.json itself, when we return the values for the key, we go back up a level by recursing. See here:

```py
# Edited for readability + extra comments.
# This is the new function that I created for generating a database. The old function is on github, but is extremely clunky and failed to match the benchmarks in speed on this one (which is surprising for python recursion).
# https://github.com/KidWizardOfTheWeb/ReaperCo.ModManager/blob/master/modfileutils.py#L134

# Recursive implementation of generate_file_DB_for_mod(). 
# Can save time, but will probably suffer on longer mods.
def generate_file_DB_for_mod_pack(curr_path):
    if os.path.isdir(curr_path):
        # if the given path is a directory, add it to the dict as a key, 
        # then update the values
        output_dict = {}
        for item in os.listdir(curr_path):
            # We want to ignore these in our base directory, as we use these ourselves and games do not.
            if item == 'db.json' or item == 'modinfo.ini':
                continue
            
            full_path = os.path.join(curr_path, item)
            if os.path.isdir(full_path):
                # Add the directory as a key, then recurse and go deeper. 
                # The value returned will be the files needed, 
                # or it'll go down another dir if needed.
                output_dict[item] = generate_file_DB_for_mod_pack(full_path)
                pass
            else:
                # If it ain't a directory, it's a file, do the ol' saving technique:
                # full_path = file path of where the file 
                # was from originally (CRITICAL TO SAVING).
                # 2nd param = the file extension. 
                # fileHash256 = the hash of the file, useful for 
                # diff checking if mods don't match with someone else.
                # We used to do these in separate lines, 
                # but I figure that I'd cut that out and lower the amount of 
                # steps to do. 
                # Problem is readability, so I don't like that. 
                # Time save must be negligible but haven't run benchmarks on it.
                with open(full_path, 'rb', buffering=0) as f:
                    fileHash256 = hashlib.file_digest(f, 'sha256').hexdigest()
                output_dict.update({item: [full_path, os.path.splitext(item)[1], fileHash256]})
                pass
        
        return output_dict
    print("DB generation finished.")
    # This one is weird, I kind of accidentally had this just work.
    # For some reason, even if the outer function has a pass, 
    # it still returns the output_dict? I think the final recurse does it but 
    # that makes no sense.
    # We use this in a wrapper function, so the file saving happens after this.
    pass
```

Even that wasn't enough, because there was a key component in the merging of databases that broke everything.

```py
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

        # KEY: THIS FOR-LOOP NEVER GOES DOWN ANY SUBFOLDERS. 
        for keys in new_dict_keys:
            combined_file_dict[keys].update(new_dict_to_add[keys])
```

Because of that, I had to rewrite *the whole thing*.

The good thing is that I was able to use `os.walk()`, which is a neat function that I used for some other processes in the mod manager before. What's even better is that, by using this function to walk every file and subdirectory, we can actually trace things back due to our original db.json schema. Remember how key-dicts were directories and key-lists were files? We can actually check the current full path of the directory we're walking and, if we ever have a KeyError, retrace our steps entirely using the root of the mod to the current path and hash properly, starting from the top.

Even more than that, python's rules about references helped out here too. 

If you have a dictionary, and set another variable equal to that dictionary, it's a shallow copy, meaning any edits to one dictionary will be done to the other. Even on sublevels. 

Thus, if we can walk the entire directory of a mod and hash through all subdirectories... we can just make a clone of our dictionary to act as a pointer object. This pointer will then update the main dictionary without destroying higher level data of that dictionary, as it'll only hash through the paths and files that have changed, and the original dictionary stays intact by not modifying that at all and letting our "pointer" do the work during the loop.

Run this for every mod and bam, full directory walking and hashmap modification.

There's more to all of this, but not enough space or time to detail all of it in this update. More features such as easy mod folder access with double clicking were added in the recent releases, which you can check out on [the github repo](https://github.com/KidWizardOfTheWeb/ReaperCo.ModManager).


### A "Game Project"?

While I did say in November that I would be starting a game project of sorts, I did NOT expect it to be this one.

Recent developments on the videogame preservation space has yielded two Sonic Riders: Zero Gravity prototypes (one for PS2, one for Wii). The key here is that **the PS2 version has full debug symbols**. This means that a decompilation will move much faster than it usually would, as most of the data structures are detailed as well.

I've learned a lot about Sonic Riders for the GameCube and Zero Gravity for the Wii in my time in the modding scene, and that's finally getting put to good use with the Zero Gravity decompilation. There are a few volunteers and progress is starting slow, but for anyone interested in the project, [check out this hyperlink and join the discord to support the project.](https://discord.gg/GvZqtydn6v)

It is a bit hard for me to work on it often, but it's definitely something I'd like to do over time.


### Timing is weird.

I ended up finishing this post in March, which there will be a separate update for. This was mainly a way to finish the work that was already done here, even though it was mainly incomplete.

Much more has happened since January, so the March post will be more up to date.