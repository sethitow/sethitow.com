---
title: "What's on Orville's screen?"
date: 2020-04-19T13:51:34-07:00
---

<figure class="image">
  <img src="media/dodo-airlines.png">
  <figcaption>Oh Orville, what might you be hiding?</figcaption>
</figure>

Inspired by [Bertrand Fan's investigation](https://tomnookslaptop.bert.org) into what Tom Nook exactly had on his laptop which (made #1 on [Hacker News](https://news.ycombinator.com/item?id=22867775)), I decided to see what Orville (the Dodo Airlines clerk) had on his screen.

<!-- <img src="media/text-message.png"> -->

This effort was made entirely possible though the amazing [hactool](https://github.com/SciresM/hactool), which does all the heavy lifting of decompressing the image file system. It's clear the maintainer and contributors have put a lot of effort into developing hactool.

First order of business was to compile hactool for macOS because only a Windows binary distribution is provided. It compiles easily on macOS with Clang `cp config.mk.template config.mk && make`. hactool requires that a keyset be available in the default location or provided through the `-k` option. 

Starting with Animal Crossing card dump in an XCI format, hactool identifies 5 partitions:
- root
	- Contains the partition map for the rest of the image
- update
	- I suspect after the original release of a game, the base image remains unchanged and updates are only provided as "overlay" images. It seems the platform uses something like OverlayFS to mount the update on top of the original game image into a single file tree. I imagine this is similar to how Docker "layers" are implemented. 
- normal
	- This partition is empty on this game image. 
- secure
	- This is where all the resource and executable files are.
- logo
	- This contains a generic splash screen animation. 

The "secure" partition is what we're interested in, and it can be dumped using hactool's `--securedir=<dir>` option. Inside the new folder, there are a number of Content Archive files (`.nca`). I'm unsure of what exactly all these .nca files are for, however the one with the largest file size is the one that we're interested in.

The NCA files are generally divided into two regions, the ExeFS and the RomFS. ExeFS contains the actual executable binaries, while RomFS contains resource files. Hactool makes quick work of extracting these .nca archives. There's lots of data inside RomFS that's extraneous to our goal, but we're interested in the `Model` directory.

```
⋊> ls romfs/Model | wc -l
37549
```

Goodness gracious, great balls of fire! How am I ever supposed to find a simple desktop computer model among 37549 files!

Fortunately, most of the files are all sensibly named in English. It took me a bit of trial and error, however the model we're after is actually not a discrete item; it's part of a model of the entire interior of the airport.

At this point, all of the models carry a `.zs` file extensions. This was one portion of the process that I couldn't find well documented, however the `file` utility tipped me off that the models are Zstandard compressed. There's a reference implementation of the Zstandard compression algorithm available for most systems `zstd -d <file>`. Once these are decompressed, we end up with a file in the SARC container format. There's not much magic in the SARC format, and I'm not sure what purpose it serves; it doesn't provide any compression or metadata, it just groups files into a single binary blob with a simple header. All the SARC files that I inspected only had one BFRES file inside of them making SARC seem like unneeded overhead.

BFRES is a general purpose media format used since 2012. It can contain models, textures, bone rigging, animations, and other data. There are lots of tools that can supposedly open the BFRES model format, however it seems that this platform (or maybe just this game) uses a slightly different BFRES format and tools designed for the Wii U were unable to open these files. The only tool I got to open the files without immediate error was [Switch Toolbox](https://github.com/KillzXGaming/Switch-Toolbox), although I wasn't able to view the 3D models directly inside Switch Toolbox. It was able to display the model skins and export the 3D portion to Collada (`.dae`) and macOS Preview was able open them just fine. 

And with that, we have our scene! 

<figure class="image">
  <img src="media/orville-computer.png">
  <figcaption>The entire interior of the airport is in one model file.</figcaption>
</figure>

<figure class="image">
  <img src="media/dodo-airlines-screen.png">
  <figcaption>Just the screen extracted from the texture.</figcaption>
</figure>
