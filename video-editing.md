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

### Determining video resolution and other details

`ffprobe` is a part of FFmpeg, and allows you to analyze a video file for details. The basic usage is:

```sh
ffprobe -v quiet -print_format json -show_format -show_streams <input.mp4>
```

This will show you _a lot_ of information, so you may want to use `grep` to pull out just the information you need. For instance, to get the width and height (resolution) of the video:

```sh
ffprobe -v quiet -print_format json -show_format -show_streams \
<input.mp4> \
| grep "\"width\"|\"height\""
```

To get the encoding information from the file (video encoding listed first, audio encoding listed second):

```sh
ffprobe -v quiet -print_format json -show_format -show_streams \
<input.mp4> \
| grep "codec"
```

And to get details on the file format:

```sh
ffprobe -v quiet -print_format json -show_format -show_streams \
<input.mp4> \
| grep "format"
```

### Converting a video format

Most videos will come to you in `.mp4` format, which is what you want, as it's the most balanced in terms of being widely (basically universally) supported while maintaining a small filesize.

If you get videos in any other format, FFmpeg should have no trouble converting them, using [`convertvid.sh`](https://github.com/rootwork/bash-scripts/blob/main/videos/convertvid.sh):

`./convertvid.sh <input.mp4>`

Note that "format" and "encoding" are not necessarily the same thing, for instance Apple Quicktime Movies (the default export from an iPhone) can be saved as `.mp4`, `.mov` or other file extensions.

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
ffmpeg \
-i <input1.mp4> -i <input2.mp4> \
-filter_complex "[0:v:0][0:a:0][1:v:0][1:a:0]concat=n=2:v=1:a=1[outv][outa]" \
 -map "[outv]" -map "[outa]" \
<output.mp4>
```

### Adjusting resolution (scaling and cropping videos)

In general you'll probably want the highest-resolution version of the video possible -- that is, the resolution you get right out of the camera. If you need to save filesize, however, or you're trying to join two videos that have different resolutions, you might need to downscale or resize one of them.

First [determine the size of a video](#determining-video-resolution-and-other-details).

To scale (keep the aspect ratio of) a video, use the following command, replacing `width` and `height` appropriately:

```sh
ffmpeg -i <input.mp4> \
-vf scale=width:height,setsar=1:1 \
<output.mp4>
```

You can [test the values using `ffplay`](#testing-advanced-video-filters).

To crop a video (for instance if you have two different aspect ratios in videos you're trying to join) first scale to the closest width or height value. Then use this command, replacing `width`, `height`, `x` and `y` (the starting positions, with `0:0` being top left):

```sh
ffmpeg -i <input.mp4> \
-filter:v "crop=width:height:x:y" \
<output.mp4>
```

Again, you can [test your values using `ffplay`](#testing-advanced-video-filters).

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
ffmpeg -i <input.mp4> \
-vf fade=t=in:st=[starttime]:d=[duration] \
-af afade=t=in:st=[starttime]:d=[duration] \
fade=t=out:st=[starttime]:d=[duration] \
-af afade=t=out:st=[starttime]:d=[duration] \
<output.mp4>
```

`[starttime]` and `[duration]` are specified in seconds. So, for instance, to set a video fade-in of 5 seconds, an audio fade-in of 2 seconds, a video fade-out of 10 seconds and an audio fade-out of 1 second you would do the following for a 64-second video (note you'll have to compute the fade-out start times yourself!):

```sh
ffmpeg -i <input.mp4> \
-vf fade=t=in:st=0:d=5 \
-af afade=t=in:st=0:d=2 \
fade=t=out:st=54:d=10 \
-af afade=t=out:st=63:d=1 \
<output.mp4>
```

You can [test your fade using `ffplay`](#testing-advanced-video-filters).

### Adding an image overlay

#### Watermarks

All Bells of the Cascades videos should have a watermark of the BOC logo. There are [several sizes of the circular logo](https://github.com/BellsOfTheCascades/helpers/tree/main/images) that you can use depending on the resolution of your video. (Higher-resolution videos need larger watermark images to fill the same amount of proportional space.) Note that we generally want to apply the watermark _after_ any fades are applied so that it is visible for the full duration of the video.

[`markvid.sh`](https://github.com/rootwork/bash-scripts/blob/main/videos/markvid.sh) is designed just for that. The script takes three parameters: The path to the video, the path to the watermark image, and the distance (in pixels) from the lower-right corner:

```sh
./markvid.sh <input.mp4> <watermark.png> PIXELDIST
```

The script will create a new video with "-marked" appended to the input filename.

For instance, to apply [`boc-logo-circular_100.png`](https://github.com/BellsOfTheCascades/helpers/blob/main/images/boc-logo-circular_100.png) to `video.mp4` with a margin of 10 pixels, you'd use this command:

```sh
./markvid.sh video.mp4 watermark.png 10
```

You will probably have to test out which watermark and margin work best for a given video a few times.

Generally watermarks should be transparent (unless you want a rectangular watermark), so PNG files work best.

#### Full-video image overlays

You may want to create an overlay that covers the full video screen (while audio is playing). To do this, first [determine the dimensions of your video](#determining-video-resolution-and-other-details). This will tell you the size of the image you need to create, which should be a PNG.

Then use FFmpeg to apply the image to the portion of the video you want to cover. For instance, to apply `slide.png` to `video.mp4` from 55 seconds to 1 minute 3 seconds, fading in for 1 second and out for 2 seconds (for a total of 8 seconds), you would use this command:

```sh
ffmpeg -i video.mp4 -loop 1 \
-t 8 \
-i slide.png \
-filter_complex "[0:v][1:v] overlay=x=(main_w-overlay_w)/2:y=(main_h-overlay_h)/2:enable=between(t\,52\,57)" \
video-with-overlay.mp4
```

You can [test your overlay using `ffplay`](#testing-advanced-video-filters).

Using FFmpeg's `main_w-overlay` and `main_h-overlay` values allows us to position the image in the center automatically, so you could also use this to overlay an image in the center that does not fully cover the video.

There are ways to fade-in and fade-out overlays, as well as applying overlays without transparency (like JPEGs) but the syntax starts to get complicated. Some examples: [1](https://stackoverflow.com/questions/38633369/how-to-add-a-fade-in-within-overlay-with-ffmpeg) [2](https://gist.github.com/sevitz/7872378), [3](https://stackoverflow.com/questions/72014088/ffmpeg-lavfi-is-it-possible-to-fade-out-an-image-overlay-that-was-loaded-with-a), [4](https://superuser.com/questions/881002/how-can-i-avoid-a-black-background-when-fading-in-an-overlay-with-ffmpeg), [5](https://superuser.com/questions/1201524/ffmpeg-overlay-image-on-video-with-fade-effect)

#### Adding semitransparent color overlays

Frequently, if you're [adding text overlays](#adding-text-overlays), you'll want some shading (or lightening) behind the text so that the text is readable. For that, use [FFmpeg's `drawbox`](https://ffmpeg.org/ffmpeg-filters.html#drawbox).

Here are some examples. You can [test different values using `ffplay`](#testing-advanced-video-filters).

Add a 50% opacity black bar across the bottom of the video:

```sh
ffmpeg -i <input.mp4> -vf "format=yuv444p,drawbox=y=ih/1.45:color=black@0.5:t=fill,format=yuv420p" <output.mp4>
```

Add a 40% opacity blue bar with the [golden ratio (phi)](https://en.wikipedia.org/wiki/Golden_ratio) height across the bottom of the video:

```sh
ffmpeg -i <input.mp4> -vf "format=yuv444p,drawbox=y=ih/PHI:color=blue@0.4:t=fill,format=yuv420p" <output.mp4>
```

The two wrapping `format` statements are to ensure the color is semitransparent regardless of the type of video.

`drawbox` can be used in conjunction with other filters such as `drawtext`, but is shown here on its own for clarity.

#### Adding text overlays

FFmpeg allows you to write text directly onto the video.

For instance, to write `www.BellsOfTheCascades.org` using the BOC font, onto `video.mp4` between 30 seconds and 40 seconds, you would use this command, where `path/to/font.ttf` is the full path to the font file on your system:

```sh
ffmpeg -i video.mp4 \
-vf "drawtext=fontfile=path/to/font.ttf:text='www.BellsOfTheCascades.org':fontcolor=white:fontsize=150:x=(w-text_w)/2:y=h-th-250:enable='between(t,46,49)'" -\
codec:a copy \
part2-faded-overlaid.mp4
```

You can play with the size (`fontsize`) and positioning (`x`, `y`) to achieve the effect you want [by using `ffplay`](#testing-advanced-video-filters).

If you want to have the text appear for the full duration of the video, leave off the `:enable='...'` portion.

Here's [a good guide on `drawtext`](https://ottverse.com/ffmpeg-drawtext-filter-dynamic-overlays-timecode-scrolling-text-credits/) and [a more complex example](https://www.ffmpegbyexample.com/examples/50gowmkq/fade_in_and_out_text_using_the_drawtext_filter/).

## Testing advanced video filters

Whenever using FFmpeg directly with a command like `ffmpeg -i <input.mp4> -vf...` or `ffmpeg -i <input.mp4> -filter_complex...`, you can test the effects of what you're doing without re-encoding the entire video by using `ffplay`.

Unfortunately, the syntax for `ffplay` is structured a little differently than `ffmpeg`, so you can't simply drop in one for the other. If you have a command like this:

```sh
ffmpeg -i <input.mp4> \
-vf "drawtext=fontfile=path/to/font.ttf:text='www.BellsOfTheCascades.org':fontcolor=white:fontsize=150:x=(w-text_w)/2:y=h-th-250:enable='between(t,46,49)'" -\
codec:a copy \
<output.mp4>
```

The corresponding `ffplay` command to test it would be:

```sh
ffplay \
-vf "drawtext=fontfile=path/to/font.ttf:text='www.BellsOfTheCascades.org':fontcolor=white:fontsize=150:x=(w-text_w)/2:y=h-th-250:enable='between(t,46,49)'" \
<input.mp4>
```

Essentially you put the filters (and any other operations) first, and move the reference to the input file to the end of the command. There is no output file; instead the video will simply play on your screen.

## Finishing up

You may want to rename, or save a copy, of the final version of your video by appending "-final" to the filename so that when you come back to it in 6 months or 6 years, you don't have to figure it out. You may also want to keep a record of the commands you run (especially custom fades, overlays, or adding text) if you're working on a video series or otherwise think you might want to have a similar-looking video in the future.

### Creating a video image or screenshot

Social media systems will generate automatic thumbnail images when you upload a video, sometimes giving you a choice. But most platforms also allow you to upload a custom image to use as your thumbnail. If you don't already have a "cover image" (such as a slide overlay), you can take your own video screenshots with [`vidcap.sh`](https://github.com/rootwork/bash-scripts/blob/main/videos/vidcap.sh).

Running it with no arguments will take an image of the video at full resolution from the exact middle (by time code):

```sh
./vidcap.sh <input.mp4>
```

If you want a selection of images to choose from, add a number to the end of the command and you'll get that many screenshots, spread evenly across the length of the video:

```sh
./vidcap.sh <input.mp4> 15
```

You can also create a montage of images, sometimes called contact sheets or screen-caps. For instance, this creates a montage of 5 images spread across the video, excluding the beginning and end (where fades might occur and thus have little image data):

```sh
./vidcap.sh -o -n <input.mp4> 5
```

### Minimizing video file size

Again, generally you want to prioritize video quality over file size, but if you want to archive things without losing too much quality, use [`minvid.sh`](https://github.com/rootwork/bash-scripts/blob/main/videos/minvid.sh), which re-encodes the video using a modern format and strips all metadata.

```sh
./minvid.sh <input.mp4>
```

You could also [directly adjust the video resolution](#adjusting-resolution-scaling-and-cropping-videos) in order to achieve a smaller file size.
