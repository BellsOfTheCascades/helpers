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

If a command specifies separate `<input.mp4>` and `<output.mp4>` values, then **do not try to overwrite the input file** by providing the same filename in both places. FFmpeg streams videos and can't overwrite a file while simultaneously reading it. You can always delete the original file once you're done.

## Basic edits

Sometimes the videos that come in need to be manipulated before you start. Here are some basic edits:

### Converting a video format

Most videos will come to you in `.mp4` format, which is what you want, as it's the most balanced in terms of being widely (basically universally) supported while maintaining a small filesize.

If you get videos in any other format, FFmpeg should have no trouble converting them, using [`convertvid.sh`](https://github.com/rootwork/bash-scripts/blob/main/videos/convertvid.sh):

`./convertvid.sh <input.mp4>`

### Rotating a video

Video (and image) rotation states can be confusing; often smartphone cameras will set a rotation position in the file's metadata without actually rotating the pixels themselves. If you're seeing media rotated on some devices or in some situations but not others, this is probably what's going on.

#### 1. Determine if rotation metadata exists, and remove it

Viewing and rewriting metadata are quick commands that do not require the entire video to be read or re-encoded. To view the rotation metadata, use the following command (`ffprobe` is a part of `ffmpeg`):

```sh
ffprobe -loglevel error -select_streams v:0 -show_entries stream_tags=rotate -of default=nw=1:nk=1 -i <input.mp4>
```

If nothing gets displayed, there is no rotation metadata and you can move on to the next step.

If there is rotation data, you'll simply see a number returned, like `90` or `180`. It's best to remove the rotation metadata to ensure the video won't show up "extra rotated" on some devices once you're done. To remove the metadata, run:

```sh
ffmpeg -i <input.mp4> -metadata:s:v rotate="0" -c copy <output.mp4>
```

#### 2. Rotate the video

The command you enter will depend on to what extent the video is rotated. FFmpeg will need to re-encode the file in order to rotate it, so this will take a few moments depending on the length of the video.

To rotate the video 90 degrees clockwise:

```sh
ffmpeg -i <input.mp4> -vf "transpose=1" <output.mp4>
```

To rotate 180 degrees clockwise:

```sh
ffmpeg -i <input.mp4> -vf "transpose=1,transpose=1" <output.mp4>
```

And to rotate 270 degrees clockwise:

```sh
ffmpeg -i <input.mp4> -vf "transpose=1,transpose=1,transpose=1" <output.mp4>
```

(`transpose` can take other values, but they can be hard to visualize. In brief, a value of `0` rotates counter-clockwise and flips the video vertically, `2` rotates counter-clockwise but does not flip, and `3` rotates clockwise and flips.)

### Trimming a video

In most situations, you'll have portions of a video you want to cut off from the beginning and end of a file. You may also want to take a single file and divide it into multiple files.

When trimming, it's important to keep two things in mind:

- **Trim to the _audio length_ that you want, not the video length.** If you are going to later add overlays or fade to black, you (generally) want the audio to continue "behind" that transition. To keep things simple, account for that extra bit when you're first trimming the file.
- **Provide enough at the beginning and end for fades.** Most videos have fade-in and fade-out of both the audio and video for a smooth effect. Whatever amount of time you expect to use (usually 2-5 seconds), make sure to account for it when you trim.

We're going to use [`trimvid.sh`](https://github.com/rootwork/bash-scripts/blob/main/videos/trimvid.sh) for the trimming, which takes four values: The input filename, the start position, the end position, and the output filename.

- Note that what the script is looking for are _start and end positions_ and not _duration of the clip_.
- The end position is optional; if omitted it will run to the end of the video.
- For the start and end positions, you can enter it as a number of seconds (e.g. `30` for half a minute; `90` for a minute and a half) or in the format `HH:MM:SS` (e.g. for 2 minutes 38 seconds it would be `00:02:38`).
- If you want to start at the beginning of the video, enter a start position of `0`.
- The output filename is optional; if omitted then it simply creates a file with "-trim" appended to the input filename.

View your video file and determine what you want the start and end positions to be. Here are some examples.

_Trim input.mp4 beginning at 1 minute, 29 seconds to the end of the video:_

```sh
./trimvid.sh <input.mp4> 00:01:29
```

_Trim input.mp4 beginning at 1 minute, 29 seconds and lasting for 90 seconds (one and a half minutes):_

```sh
./trimvid.sh <input.mp4> 00:01:29 90
```

_Trim input.mp4 beginning at the start of the video, ending at 1 hour, 52 minutes, 56 seconds, and name the video "final.mp4":_

```sh
./trimvid.sh <input.mp4> 0 01:52:56 <final.mp4>
```

### Joining videos

Sometimes you have multiple clips you want to combine into one.

Note that generally it's useful to apply any fades to the individual clips _before_ joining, as it's much trickier to do fades in the middle of a video. Additionally, while it's possible to join videos in different orientations (portrait vs. landscape) or different sizes/resolutions, some video players and social media will react strangely so it's not recommended (see [adjusting resolution](#adjusting-resolution-scaling-and-cropping-videos) for ways to normalize videos).

Use [`joinvid.sh`](https://github.com/rootwork/bash-scripts/blob/main/videos/joinvid.sh) by passing a list of videos to be joined. The videos can be in any file format. The resulting file will be named `joined.mp4`.

For example, joining two videos:

```sh
./joinvid.sh <input1.mp4> <input2.mp4>
```

or joining five:

```sh
./joinvid.sh <input1.mp4> <input2.mp4> <input3.mp4> <input4.mp4> <input5.mp4>
```

#### Troubleshooting

If joining videos doesn't work -- if you get error messages, or the joined video appears to only contain one of the videos -- you may have videos at differing resolutions. See [adjusting resolution](#adjusting-resolution-scaling-and-cropping-videos) for help with this.

You can try using [FFmpeg's concat protocol with intermediate files](https://trac.ffmpeg.org/wiki/Concatenate), but it doesn't always work and may still result in a joined video that doesn't process well on social media.

More successfully, you can use [FFmpeg's 'Concatenation of files with different codecs'](https://trac.ffmpeg.org/wiki/Concatenate) -- and remember that filename doesn't necessarily reflect codec, so even two `mp4` files can be encoded differently. The videos will need to have the same resolution. For instance, this command will work to join two videos with different encodings:

```sh
ffmpeg -i <input1.mp4> -i <input2.mp4> -filter_complex "[0:v:0][0:a:0][1:v:0][1:a:0]concat=n=2:v=1:a=1[outv][outa]" -map "[outv]" -map "[outa]" <output.mp4>
```

### Adjusting resolution (scaling and cropping videos)

In general you'll probably want the highest-resolution version of the video possible -- that is, the resolution you get right out of the camera. If you need to save filesize, however, or you're trying to join two videos that have different resolutions, you might need to downscale or resize one of them.

First, to determine the size of a video, use `ffprobe` (part of FFmpeg):

```sh
ffprobe -v quiet -print_format json -show_format -show_streams <input.mp4> | grep "\"width\"|\"height\""
```

To scale (keep the aspect ratio of) a video, use the following command, replacing `width` and `height` appropriately:

```sh
ffmpeg -i <input.mp4> -vf scale=width:height,setsar=1:1 <output.mp4>
```

To crop a video (for instance if you have two different aspect ratios in videos you're trying to join) first scale to the closest width or height value. Then use this command, replacing `width`, `height`, `x` and `y` (the starting positions, with `0:0` being top left):

```sh
ffmpeg -i <input.mp4> -filter:v "crop=width:height:x:y" <output.mp4>
```

For details about cropping with FFmpeg, [see this excellent guide](https://www.linuxuprising.com/2020/01/ffmpeg-how-to-crop-videos-with-examples.html).

## Video effects

### Fading in and out of a video

Fading involves both the video and audio of a file. While it's possible to fade each separately and to do different amounts of fade at the beginning and end, the most straightforward approach is to apply the same amount to both audio and video at both the start and end.

[`fadevid.sh`](https://github.com/rootwork/bash-scripts/blob/main/videos/fadevid.sh) takes a filename, and optionally an amount of time for the fade.

If you don't provide an amount of time, the script will ask you:

```sh
./fadevid.sh <input.mp4>
> Video length is 122.029413 seconds. How long do you want each fade to last? (in seconds)
5
> [ffmpeg reports conversion progress]
> Done. Video created at input-faded.mp4
```

But if you already know how long you want the fade to be, you can provide that amount in seconds, with the filename:

```sh
./fadevid.sh --time=5 <input.mp4>
> [ffmpeg reports conversion progress]
> Done. Video created at input-faded.mp4
```

FFmpeg will need to re-encode both video and audio to apply the fades, so this will take some time. The output video will automatically be the name of the input video with "-faded" appended.

#### Fine-tuning fades

If you want to apply differing amounts of fades to the audio and video, and/or to the start and end of the clip, you can use FFmpeg directly. The syntax is:

```sh
ffmpeg -i <input.mp4> -vf fade=t=in:st=[starttime]:d=[duration] -af afade=t=in:st=[starttime]:d=[duration] fade=t=out:st=[starttime]:d=[duration] -af afade=t=out:st=[starttime]:d=[duration] <output.mp4>
```

`[starttime]` and `[duration]` are specified in seconds. So, for instance, to set a video fade-in of 5 seconds, an audio fade-in of 2 seconds, a video fade-out of 10 seconds and an audio fade-out of 1 second you would do the following for a 64-second video (note you'll have to compute the fade-out start times yourself!):

```sh
ffmpeg -i <input.mp4> -vf fade=t=in:st=0:d=5 -af afade=t=in:st=0:d=2 fade=t=out:st=54:d=10 -af afade=t=out:st=63:d=1 <output.mp4>
```

### Adding an image overlay

#### Watermarks

All Bells of the Cascades videos should have a watermark of the BOC logo. There are [several sizes of the circular logo](https://github.com/BellsOfTheCascades/helpers/tree/main/images) that you can use depending on the resolution of your video. (Higher-resolution videos need larger watermark images to fill the same amount of proportional space.)

[`markvid.sh`](https://github.com/rootwork/bash-scripts/blob/main/videos/markvid.sh) is designed just for that. The script takes three parameters: The path to the video, the path to the watermark image, and the distance (in pixels) from the lower-right corner:

```sh
./markvid.sh <input.mp4> <watermark.png> PIXELDIST
```

For instance, to apply [`boc-logo-circular_100.png`](https://github.com/BellsOfTheCascades/helpers/blob/main/images/boc-logo-circular_100.png) to `video.mp4` with a margin of 10 pixels, you'd use this command:

```sh
./markvid.sh video.mp4 watermark.png 10
```

You will probably have to test out which watermark and margin work best for a given video a few times.

Generally watermarks should be transparent (unless you want a rectangular watermark), so PNG files work best.

