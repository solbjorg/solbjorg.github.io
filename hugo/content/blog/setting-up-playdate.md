+++
title = "Setting Up a Playdate Dev Environment on Linux"
date = "2026-03-12T20:36:17+01:00"

tags = ["tech","gamedev","playdate",]
+++

I've been wanting to get back into gamedev for some time, and I think the key to doing this is [to make some small games](https://farawaytimes.blogspot.com/2023/02/how-to-make-good-small-games.html). A fun platform for this that I've been eyeing for some time is the [Playdate](https://play.date). I don't own one, as getting one shipped to Norway is surprisingly exxpensive, but do have access to one on occassion and I was hoping I could mostly get by with the simulator (which seems to mostly be true!). This article will go into the process of getting a decent developer environment set up, which I hope to follow up with updates on the gamedev process itself.

It should also be noted that I'm doing this on Arch Linux, though I think most of what I did here is easy to transfer to any other distro.

## Setting up the SDK and type hints

This part was fairly straightforward! I ended up collapsing it into a simple `setup.sh` script so that this could easily be replicated on another machine:

```bash
#!/bin/bash
set -euo pipefail

if [ ! -d "PlaydateSDK" ]; then 
	echo "Setting up Playdate SDK"
	wget https://download.panic.com/playdate_sdk/Linux/PlaydateSDK-latest.tar.gz
	tar xvf ./PlaydateSDK-latest.tar.gz
	mv PlaydateSDK{-3.0.3,} # normalize folder name
	rm ./PlaydateSDK-latest.tar.gz
fi
if [ ! -d "playdate-types" ]; then 
	echo "Setting up Playdate types"
	git clone git@github.com:balpha/playdate-types.git 
	ln -s $(dirname "$0")/playdate-types/__types.lua $(dirname "$0")/PlaydateSDK/CoreLibs/__types.lua
	mv PlaydateSDK/CoreLibs/__stub.lua{,.bak}
fi
```

All this does is get the official SDK, extract the tar, and then set up [playdate-types](https://github.com/balpha/playdate-types) with a symlink.[^1] This hardcodes the version number to be 3.0.3---I could make this neater, but for a simple setup script that I run once and then forget I decided this was good enough. The SDK README also suggests that you run its setup script, but all it does is add an icon to your desktop (which I don't have anyway) and add a (fairly aggressive) udev rule. As I do not have a desktop as such and I'm not yet working with real hardware it can be skipped.

## Adding good support for the Lua LSP in Neovim

I'm not an experienced Lua developer in the slightest, but after some searching I found [this post](https://devforum.play.date/t/neovim-central-thread/7953/7) by Mario Carballo on the Playdate forums giving instructions on how to get this working elegantly for neovim. (It seems to be mostly the same for VSCode, however!) 

Basically, I simply had to place a file `.luarc.json` in the root of the project telling it where to find the library I wanted typing from. I tweaked the answer slightly and ended up with this:

```json
{
  "telemetry.enable": false,
  "runtime.version": "Lua 5.4",
  "runtime.special": {
    "import": "require"
  },
  "runtime.nonstandardSymbol": ["+=", "-=", "*=", "/="],
  "diagnostics.globals": [
    "playdate",
    "json"
  ],
  "diagnostics.disable": ["redefined-local"],
  "diagnostics.neededFileStatus": {
    "codestyle-check": "Any"
  },
  "diagnostics.libraryFiles": "Disable",
  "completion.callSnippet": "Replace",
  "workspace.library": ["PlaydateSDK/CoreLibs"],
  "workspace.ignoreDir": ["Source/external"]
}
```

After adding this I got nice type annotations and docs in Neovim for free (since I already had a Lua LSP set up for editing my Neovim config).

## Setting up the simulator to run our "game"

I immediately knew I'd like to abstract this into a `build.sh` script that compiled the game and then ran it in the simulator for me. Here it is:

```bash
#!/bin/bash
set -euo pipefail

SCRIPT_PATH=$(dirname "$0")
export PLAYDATE_SDK_PATH=$SCRIPT_PATH/PlaydateSDK
PlaydateSDK/bin/pdc $SCRIPT_PATH/Source $SCRIPT_PATH/Game.pdx
PlaydateSDK/bin/PlaydateSimulator $SCRIPT_PATH/Game.pdx/
```

`pdc` is the compiler that ships with the SDK. By default opening the simulator requires you to either manually select the game folder or drag it into the simulator, which is a bit of a hassle to do when rapidly iterating. Luckily it can also take the path to the game as an argument and open it directly that way. After running the build.sh script, this shows up quite quickly:

{{< resize-image src="/post/setting-up-playdate/simulator.png" height="400" alt="The Playdate simulator running next to console output" >}}

PlaydateSimulator is however built with dynamic linking, so you may have to install some libraries to get it to work---this isn't really documented as far as I could see, so Linux users are on their own. I only had to run `sudo pacman -Syu webkit2gtk-4.1` to get it to work on my machine, but after a quick look with `ldd` it relies on far more libraries than that.[^3]

## Making a basic Playdate game

Here I simply followed [the official guide](https://sdk.play.date/3.0.3/Inside%20Playdate.html). In the end the structure looks something like this:

```
.
├── .luarc.json
├── PlaydateSDK/
├── playdate-types/
├── setup.sh
├── build.sh
└── Source/
    ├── main.lua
    ├── pdxinfo
    └── images/
        └── ball.png
```

(More on the build script in the next section.) This does strike me as a little messier than I would've liked---I will probably end up amending the setup script to place the SDK and type hints elsewhere.[^2]

Just for testing, I ended up with a very simple `main.lua` file adapted from the official guide linked above:

```lua
import "CoreLibs/object"
import "CoreLibs/graphics"
import "CoreLibs/sprites"

local gfx <const> = playdate.graphics

local playerImage = gfx.image.new("images/ball.png")
assert(playerImage)

local playerSprite = gfx.sprite.new(playerImage)

playerSprite:moveTo(200, 120)
playerSprite:add()

function playdate.update()
    if playdate.buttonIsPressed(playdate.kButtonUp) then
        playerSprite:moveBy(0, -2)
    end
    if playdate.buttonIsPressed(playdate.kButtonRight) then
        playerSprite:moveBy(2, 0)
    end
    if playdate.buttonIsPressed(playdate.kButtonDown) then
        playerSprite:moveBy(0, 2)
    end
    if playdate.buttonIsPressed(playdate.kButtonLeft) then
        playerSprite:moveBy(-2, 0)
    end

    gfx.sprite.update()
end
```

For this test I used `ball.png` found in the SDK at `PlaydateSDK/Examples/Single File Examples/assets/ball.png`. I also added `pdxinfo` which contains metadata for the project.

## Result

After a bit of fiddling, I now have a setup script I can take with me to set the project up almost instantly on another machine, and a build script that very quickly lets me check if my code works in the simulator. I also have good type hints and documentation right in my editor. 

What I *don't* have, yet, is an impressive game---or even a game at all! Nor do I have a way to test it on real hardware, as I will have to borrow a Playdate to test that out. I'm hoping to have a followup on that sooner rather than later.

[^1]: Setting up proper typing is not strictly necessary, but I find developing anything without typing to be needlessly difficult. I'm also not entirely sure why this doesn't just ship with the SDK.
[^2]: I'm also, strictly on an aesthetic level, not a fan of `Source` rather than `src`, but it's what the examples in the SDK used, and I figure it is better to follow convention than to do whatever I prefer every time.
[^3]: I guess I'm simply lucky to have ended up with enough random junk on my machine over time that it wasn't a big burden... :)
