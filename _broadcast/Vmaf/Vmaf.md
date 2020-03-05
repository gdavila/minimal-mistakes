---
title: Simplifying the way to use vmaf
excerpt: Simplifying the way to use vmaf
---


[VMAF](https://github.com/Netflix/vmaf) has become one of the [most used tools](https://netflixtechblog.com/vmaf-the-journey-continues-44b51ee9ed12) for video quality assessment (VQA). In practise, VMAF analysis requires two inputs: (i) a  sample of the Source (Reference Video) and (ii) a sample of the transcoded video, tipically captured at the output of the encoder (Distorted Video).

One of the tools where VMAF is available is `libvmaf`, a `c/c++` library implemented in `ffmpeg`  that doesn't require uncompressing the video samples.

`ffmpeg`  usage example for VMAF computation:

```bash
ffmpeg -i <main> -i <reference> -lavfi "libvmaf=model_path=/usr/local/share/model/vmaf_v0.6.1.pkl" -f null -
```

However, VMAF requires full frame-to-frame synchronization between samples (Reference and Distorted). This fact could be a  pain in the neck, mainly for live content analysis —where is not common to get the source and the transcoded video samples synced. The most typical reasons for this desynchronization are:

* Interlaced sources and Progressive outputs, *i. e.,* Sources at 29.97i fps and Outputs at 29.97p fps.

* Offset between Source and Output video samples, *i. e.,* the Reference video sample was captured a couple of seconds either  after or before the Output video sample was taken.

* Frame rate conversion between Sources and Outputs.

Additionally, VMAF needs that Reference and Distorted videos have the same resolution. In this way a sort *normalization* between source and reference is needed. Through this blog entry, I will go over some suggestions to deal with these issues. Finally, the techniques described here are freely available on [easyVmaf](https://github.com/gdavila/easyVmaf). A python based script that allows video scaling, video deinterlacing, video syncing, etc.

## Syncing Reference and Distorted video samples

The syncing could be done by using a sample of the distorted video and sliding it frame-by-frame forward and backward in order to look up the best PSNR in regard to the Reference video. Once the best PSNR is found, the amount of time slided is the offset needed to get the frames synced.

   ```
   +-----------------------------------------------------------+
   |                                                           |
   |                     REFERENCE VIDEO                       |
   |                                                           |
   +-----------------------------------------------------------+
                        ^
                        |
                        |PSNR computation
                        |
                        v
               +-----------------+
               |   DISTORTED     |
   <--backward--   VIDEO SAMPLE  --------forward------------->
               |                 |
               +-----------------+
                  sliding window

                  ---max-\
            -----/      --\
      ------/               --\    PSNR             /\-\
   ---/                         ------------/\-----/    \-----

   <-------offset----->
   ```

The sliding windows and the PSNR computation could be done through `trim` and `psnr` (filters of `ffmpeg`), example:

```bash
ffmpeg  -i <source> -i <distorted> -lavfi "[1:v]trim=start=<OFFSET>:duration=<WINDOW_SIZE>,setpts=PTS-STARTPTS[distorted];[0:v][distorted]psnr=stats_file=psnr.log" -f null -
```

The previous command should be run by incrementing the `<OFFSET>` in steps of 1/fps in order to slide frame-by-frame the distorted video. Additionally, distored video is  trimmed at a sample of `<WINDOW_SIZE>` duration in order to simplify the PSNR computation. The `<OFFSET>` `T` that produces the max PSNR will be used to compute VMAF, i.e., :

```bash
ffmpeg -i <main> -i <reference> -lavfi "[1:v]trim=start=<T>,setpts=PTS-STARTPTS[ref];[0:v][ref]libvmaf=model_path=/usr/local/share/model/vmaf_v0.6.1.pkl" -f null -
```

The above example considers that by computing the PSNR through a sliding window in the main video, the best value was finding at `T` seconds from the start of the reference video. In this way, in order to get VMAF properly synced, the computation  should start with the Frame located at `<T>` seconds within the reference video (this is done by trim filter). This means that the frame at `<T>` seconds  of the Reference sample matches with the first frame of the Distorted sample.

## Interalaced Sources

Most live sources are in interlaced formats, i. e., 29.97i / 30i and then they are tipically transcoded to progresive format for ABR delivery i. e., 29.97p / 30p.

In order to use VMAF to compare progresive outputs with interlaced references, deinterlace is needed. `FFmpeg` allows deinterlacing by using `yadif` filter.

* **Reference n [fps] interlace —> Distorted n [fps] progressive:** deinterlacing has to be done by producing one output frame for each input frame i. e., `yadif=0:-1:0` (30i —> 30p). Example:
  
    ```bash
   ffmpeg -i <main@30i> -i <reference@30p> -lavfi "[1:v]yadif=0:-1:0[ref];[0:v][ref]libvmaf=model_path=/usr/local/share/model/vmaf_v0.6.1.pkl" -f null -
   ```

* **Reference n [fps] interlace —> Distorted 2xn [fps] progressive:** deinterlacing has to be done by producing one output frame for each input field i. e., `yadif=1:-1:0` (30i —> 60p). Example:
  
    ```bash
   ffmpeg -i <main@30i> -i <reference@60p> -lavfi "[1:v]yadif=1:-1:0[ref];[0:v][ref]libvmaf=model_path=/usr/local/share/model/vmaf_v0.6.1.pkl" -f null -
   ```

## Framerate adaptation

Although it is less common, sometimes the framerate of the distorted video could not match with the frame rate of the Reference. In this case, the framerate adaptation could be done with `fps` filter. It is recommended to always adapt the framerate of the distorted video only, not reference. Example:

```bash
ffmpeg -i <main@15i> -i <reference@30i> -lavfi "[1:v]fps=fps=30[ref];[0:v][ref]libvmaf=model_path=/usr/local/share/model/vmaf_v0.6.1.pkl" -f null -
```

## Resolution Adaptation

Resolution adaptation could be done by `scale` filter by using `bicubic` algorithm. Example:

```bash
ffmpeg -i <main@720p> -i <reference@1080p> -lavfi "[0:v]scale=1920:1080:flags=bicubic[main];[main][1:v]libvmaf=model_path=/usr/local/share/model/vmaf_v0.6.1.pkl" -f null -
```
