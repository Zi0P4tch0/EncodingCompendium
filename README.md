# EncodingCompendium

My encoding notebook - writing things down for reference.

# Software Requirements

- ffmpeg
- MediaInfo

# Video Encoding

## AVC

### Encoding 1080p Remux to 1080p AVC (reference quality)

- Video Codec: H264 (CPU)
- FPS: Same as source
- Constant Quality: RF 16 (18 seems to be the golden quality standard, but darker scenes might still suffer with it)
- Preset: VerySlow
- Profile: High
- Level: 4.1
- Tune: film
- Advanced Parameters: <code>deblock=-2,-2</code>

## HEVC 10-bit

### Preset choice and rationale behind parameters

8-bit format is not being used at all, as it introduces a noticeable color banding on SDR sources.

On a Ryzen 3700X (8c/16t) the most viable preset is <code>medium</code>. It encodes with a speed of <code>1.0x</code>.

Slower presets / higher CRF are not worth pursuing, as the encoding time inflates too much, and the quality improvement is debatable.

<code>aq-mode=2</code> seems to yield the best result (both visually and according to SSIM).</br>
<code>aq-mode=1:aq-strength:1</code> is decent too, and it consumes less bitrate than <code>aq-mode=2</code>.

<code>qcomp=0.8</code> influences CRF - this value seems to be the community standard for when trying to replicate AVC quality on HEVC and CRF is <= 23.

<code>no-sao=1:no-strong-intra-smoothing=1</code> are a MUST to preserve quality and avoid blurriness.

### Encoding 1080p Remux to 1080p HEVC-10bit

This script:

- Encodes the video track to HEVC 10-bit (SDR).
- Auto-crops the video.
- Copies the chapters metadata.
- Creates three audio tracks from the primary one (assuming it's high quality - 7.1 - 24bit - 48kHz)
    - Stereo track @ 192kbps
    - 5.1 track @ 384kbps
    - 7.1 track @ 512kbps
- Copies the first subtitle track over.

:computer:
 
    PRESET="medium"
    CRF="18"
    COLOR="colorprim=bt709:transfer=bt709:colormatrix=bt709:range=limited"
    AQ="aq-mode=2"
    CTU="ctu=32:max-tu-size=16"
    DEBLOCK="deblock=-2,-2"
    QCOMP="qcomp=0.8"
    SAO="no-sao=1:no-strong-intra-smoothing=1"
    PROFILE="main10"
    LEVEL="level-idc=41"
    EXTRA="merange=44:qg-size=16:rc-lookahead=48:keyint=240:min-keyint=24"

    CROP=$(ffmpeg -i "$1" -t 600 -vf cropdetect -f null - 2>&1 | awk '/crop/ { print $NF }' | tail -1)

    ffmpeg -i "$1" \
        -map 0:0 -map 0:1 -map 0:1 -map 0:1 -map 0:2 -map_chapters 0                                                    \
        -vf $CROP                                                                                                       \
        -c:v libx265 -preset $PRESET -profile:v $PROFILE -crf $CRF -pix_fmt yuv420p10le                                 \
        -x265-params "$COLOR:$AQ:$CTU:$DEBLOCK:$SAO:$QCOMP:$LEVEL:$EXTRA"                                               \
        -c:a:0 libopus -ac:a:0 2 -b:a:0 192K -metadata:s:1 title="English / Opus / Stereo / 24 bit / 48kHz / 192kbps"   \
        -c:a:1 libopus -ac:a:1 6 -b:a:1 384K -metadata:s:2 title="English / Opus / 5.1 / 24 bit / 48kHz / 384kbps"      \
        -c:a:2 libopus -b:a:2 512K -metadata:s:3 title="English / Opus / 7.1 / 24 bit / 48kHz / 512kbps"                \
        -c:s:0 copy                                                                                                     \
        output.mkv
### Encoding 2160p HDR10 Remux to 1080p HEVC-10but

Same as above, but we apply a <b>lanczos</b> scale video filter.

<code>-vf=scale=1920:800 -sws_flags lanczos</code>

See also HDR flags in the section below. HDR -> SDR tonemapping is totally not worth it.

### Encoding 2160p HDR10 Remux to 2160p HEVC-10bit

- Video Codec: H265-10Bit (CPU)
- FPS: Same as source
- Constant Quality: **EXPERIMENTING**
- Preset: **EXPERIMENTING**
- Profile: Main10
- Level: 5.1
- Tune: none
- Advanced Parameters: <code>aq-mode=1:aq-strength=1.0:deblock=-2,-2:qcomp=0.8:no-sao:transfer=smpte2084:colorprim=bt2020:colormatrix=bt2020nc:chromaloc=2:hdr:hdr-opt:max-cll=724,647:master-display=G(13250,34500)B(7500,3000)R(34000,16000)WP(15635,16450)L(10000000,1)</code>
    - <code>transfer=smpte2084:colorprim=bt2020:colormatrix=bt2020nc:hdr:hdr-opt</code> HDR
    - <code>chromaloc=2</code> check this on the source file with MediaInfo, make sure it's the same
    - <code>max-cll=724,647</code> check this on the source file with MediaInfo, make sure it's the same
    - <code>master-display=G(13250,34500)B(7500,3000)R(34000,16000)WP(15635,16450)L(10000000,1)</code> P3 display specification

# Compare two inputs with identical number of frames

[SSIM](https://ece.uwaterloo.ca/~z70wang/research/ssim/):

<code>ffmpeg -i main.mkv -i reference.mkv -lavfi ssim -f null -</code>

Luminance difference (**Y**) seems to be the most noticeable.

## Interpret SSIM and PSNR

https://forum.doom9.org/showthread.php?p=1334145#post1334145

> ssim: ((1-oldssim)/(1-newssim) - 1)*100 = % improvement
>
> psnr: (new - old) / 0.05 = % improvement 

# Known bugs

## Too many packets buffered for output stream 0:1

https://trac.ffmpeg.org/ticket/6375

Solution:
<code>-max_muxing_queue_size 1024</code>

Some users report that using <code>9999</code> also worked.

## libopus - Invalid channel layout 5.1(side) for specified mapping family -1

https://trac.ffmpeg.org/ticket/5718

FFmpeg is not remapping channels for <code>5.1(side)</code> streams.<br/>
A simple audio filter can fix that: <code>-af "channelmap=channel_layout=5.1"</code>
