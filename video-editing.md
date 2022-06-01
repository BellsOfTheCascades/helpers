# Video editing: Trims, fades and overlays

## Tools

We're using the [video scripts from this repo](https://github.com/rootwork/bash-scripts), which run on the command-line (Bash, though mostly tested in Z Shell) and should work fine on Linux, Mac, or Windows Subsystem for Linux.

All require [FFmpeg](https://ffmpeg.org/) to be installed. Some also take advantage of [Imagemagick](https://imagemagick.org/), which you may already have installed (try running `magick --version` at the command line; any version of 6.x or newer will be fine).

In some cases, we simply run FFmpeg or Imagemagick commands directly.

## Starting out

I find it helpful to organize videos first by season, then by concert or rehearsal. For instance `Videos` > `2022 Spring` > `2022-05-17 Rehearsal`. You may also want to rename the video files with the name of the piece(s) being performed, if appropriate.

**Highly recommended:** Gather all your raw video files into a subfolder of your current project called `originals`. Copy videos out of this folder for editing, leaving the originals in place in case you ever need to go back to square one.

Finally, you may find you need to slightly tweak some of the commands below for a given project. It can be a lifesaver to keep copies of the commands you use in a file as you work, so that if you discover you have to start from the beginning again, you have ready-to-execute commands for each step.

### A note on command syntax

When you see something in a command between angle brackets, `<like this>`, that means it's meant to be a filename that you provide. For instance, you'll see many commands that reference `<input.mp4>` and `<output.mp4>`. These aren't files with those literal names, they mean you should replace them with your current (input) file and your preferred (output) file.

Don't include the angle brackets in the command itself. So if a command looks like this:

```sh
script.sh -i <input.mp4> -o <output.mp4>
```

and you have a file called `video-22.mp4`, and want to end up with a new file called `video-22_edited.mp4`, you would enter the command like this:

```sh
script.sh -i video-22.mp4 -o video-22_edited.mp4
```

## Basic edits

Sometimes the videos that come in need to be manipulated before you start. Here are some basic edits:

### Converting a video format

Most videos will come to you in `.mp4` format, which is what you want, as it's the most balanced in terms of being widely (basically universally) supported while maintaining a small filesize.

If you get videos in any other format, FFmpeg should have no trouble converting them, using [`convertvid.sh`](https://github.com/rootwork/bash-scripts/blob/main/videos/convertvid.sh):

`convertvid.sh <input.mp4>`

### Rotating a video

Video (and image) rotation states can be confusing; often smartphone cameras will set a rotation position in the file's metadata without actually rotating the pixels themselves. If you're seeing media rotated on some devices or in some situations but not others, this is probably what's going on.

This command will rotate the video 90 degrees clockwise:

```sh
ffmpeg -i <input.mp4> -vf "transpose=1" <output.mp4>
```