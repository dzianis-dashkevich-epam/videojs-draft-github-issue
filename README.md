## Description

(***Note: there is a lot of text on purpose. Please read to the end. Eventually, it will draw you a picture of the issue.***)

Hello, Video.Js and VHS Teams!

I am working with STB and we are using Video.Js for HLS live playback.
I noticed interesting behavior during discontinuity:

The player freezes at the last video frame of the last segment of the previous media sequence.
Audio frames from the next media sequence overlap this frozen frame.
Video frames from the next media sequence do not start.

After ~1 second: the player seeks back, buffering occurs, and the next media sequence plays fine:
Audio frames start over in sync with video frames from the next media sequence.

The most interesting part is that this issue occurs only on STB's browser( which is Chromium v72) and does not occur on my laptop's browser (which is the latest Chrome).

So I started my investigation:

I collected Video.Js debug logs from STB and my laptop and compared them:

The only difference I found is that on STB it triggers `vhs-unknown-waiting` event at some point. I checked `PlaybackWatcher` source code and found out that it recursively checks the current time each 250ms. In case if current time stays the same - `consecutiveUpdates` variable is increased. When `consecutiveUpdates` reaches its limit (for some reason it is 5 from the source code). The Player tries to recover and seeks to the last reported current time.

Well, I decided to check behavior with an increased `consecutiveUpdates` limit to bypass this seek-back:

Result: after 2 seconds of freezing, the player continues playback of the next program.

At this point, I decided to check other players and their implementations.
Here is a small comparison table based on my brief source code dive:

| Player   | Transmuxer                                                                                                                                                        | Keeps Original DTS/PTS                        | Does TimestampOffset in use for alignment?                                                                                  |
|----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Video.Js | Uses mux.js library with the following options: `keepOriginalTimestamps: true` and `remux: true`.  The Transmuxer produces both audio and video mp4.                    | Does not override source timestamps. | Yes, Video.Js uses the source buffer's TimestampOffset in order to align the source timeline with its own time. |
| Shaka    | Uses mux.js library with the following options: `keepOriginalTimestamps: true` and `remux: false`.  The Transmuxer produces only one mp4 with combined audio and video. | Does not override source timestamps.    | No, Shaka uses the same timeline values as in the source. Thus no alignment is needed.                               |
| Hls.Js   | Uses its own transmuxer implementation.  The Transmuxer produces both audio and video mp4.                                                                        | Overrides source timestamps.           | No, Hls.Js overrides source timestamps with its own timeline values (based on `buffered.end`).                                             |

Shaka and Hls.Js work fine, so it looks like something is wrong with a timestamp offset approach.

To prove my theory, I implemented my custom player based on mux.js library with Hls.js approach - override timestamps during transmuxing with use of `baseMediaDecodeTime` option and `buffered.end`.

Result: it works fine, without any freezes.

But why isn't it working with timestamp offset?
To answer this question, I decided to take a look into frames:

`ffprobe -v quiet -print_format json -show_format -show_streams -show_frames -show_packets -count_frames -count_packets "$session_mp4_video_segment_input" > "$ffprobe_video_mp4_output"`

`ffprobe -v quiet -print_format json -show_format -show_streams -show_frames -show_packets -count_frames -count_packets "$session_mp4_audio_segment_input" > "$ffprobe_audio_mp4_output"`

`ffprobe -v quiet -print_format json -show_format -show_streams -show_frames -show_packets -count_frames -count_packets "$session_ts_segment_input" > "$ffprobe_ts_output"`

Based on the reports from `ffprobe` 
I noticed, that buffered ranges reported from Chromium on STB are based on DTS, 
but buffered ranges reported from Chrome on my laptop are based on PTS.

I replaced `segmentInfo.timingInfo.start`(which is frame's PTS value from the transmuxer) 

to `segmentInfo.segment.videoTimingInfo.transmuxedDecodeStart` (which is frame's DTS value from the transmuxer) 

in [segmentLoader timestamp offset calculation](https://github.com/videojs/http-streaming/blob/main/src/segment-loader.js#L2908). Finally it works.

But why is it working with DTS-based timestamp Offset and not working with PTS-based timestamp Offset?

To answer this question I decided to draw several segments from the stream:
![image](https://user-images.githubusercontent.com/94862693/149061494-aeccd77d-f2c0-42b5-90c6-a35783e75008.png)

And how different types of frames are encoded and reference each other's macroblocks:
![image](https://user-images.githubusercontent.com/94862693/149061787-c5a22e88-849e-4b96-b2a2-c525db4230e1.png)


I did not dive really deep into chromium source code, and it was not possible to collect chromium debug logs from STB.

But I noticed an interesting flag in the old chromium revision:

https://chromium.googlesource.com/chromium/src/+/refs/tags/72.0.3626.121/media/base/media_switches.cc#293

Based on the information above, I made the following assumption:

It looks like after updating timestamp offset and appending the next frames browser does not hit the very first I-Frame, and because of this - it is not possible to decode further frames in the closed GOP. That is why the browser drops all frames until the next [random access point](https://www.w3.org/TR/media-source/#random-access-point) - in our case next I-Frame. Because the stream is 30 fps and because it is based on Apple's recommendation:
[1.13. Key frames (IDRs) SHOULD be present every two seconds.](https://developer.apple.com/documentation/http_live_streaming/hls_authoring_specification_for_apple_devices)
I saw this 2 seconds of freeze. (the root cause of the `vhs-unknown-waiting` from the beginning.)

In order to prove my assumption (even without debug logs from the chromium), I decided to re-encode the original stream with frame info overlay on it:

`ffmpeg -i $input -vf "drawtext=fontfile=Arial.ttf:text='Frame\: %{frame_num} Type\: %{pict_type} PTS\: %{pts}':start_number=0:x=(w-text_w)/2:y=(h-text_h)/2:fontcolor=white:fontsize=21:box=1:boxcolor=black@0.75:boxborderw=6" -c:v libx264 -c:a copy -copyts -muxdelay 0.01568455555555555555 -map_metadata 0 -force_key_frames:v source -force_key_frames:a source -r 30000/1001 $output`

I tested this stream both using PTS-based offset and DTS-based offset:

with PTS-based offset -> I see that after 2 seconds of freeze it starts the next program with frame #60 which is the first I-Frame of the 2nd closed GOP.
![image](https://user-images.githubusercontent.com/94862693/149062830-7ef352f7-d47f-4b5a-b1f8-ee083877e47d.png)

with DTS-based offset -> I see no freezes and it starts the next program with frame #0 which is the first I-Frame of the 1st closed GOP.
![image](https://user-images.githubusercontent.com/94862693/149062988-39149aba-61fa-4cf6-a8ff-b2d59ee78e14.png)

My Proposal is to add a config option to choose between PTS-based and DTS-based offset calculations, in order to support DTS-based buffered browsers, such as chromium v72.

## Sources
I can provide you with two links - original source and with debug overlay by e-mail.

## Steps to reproduce
- I was testing with Video.Js v7.14.3 (VHS v2.9.2) and v 7.18.0 (VHS v2.13.1).
- Run provided stream at any DTS-based buffered browsers (eg: chromium v72).
- Wait for a discontinuity.


### Browsers

Chromium 72.0.3626.121

### Platforms

Linux

### Other Plugins

No.
