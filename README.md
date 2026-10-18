# FFmpeg with an AC-4 audio decoder

This is FFmpeg 4.3 (a development snapshot from early 2021) with an **AC-4
audio decoder**: Dolby's audio format used by ATSC 3.0 ("NextGen TV")
broadcasts, some DVB services and streaming. The decoder,
`libavcodec/ac4dec.c`, is Paul B Mahol's (2019); AC-4 is also read from raw
`.ac4` files, MP4 and MPEG-TS.

It is the FFmpeg behind a Kodi build that plays AC-4 sound.

## Kodi 21.3 and FFmpeg 6.0.1

The decoder has been ported to FFmpeg 6.0.1, the FFmpeg that Kodi 21.3
"Omega" uses, and a Kodi 21.3 for Windows with AC-4 is available at
[ferrellsl/kodi-ac4](https://github.com/ferrellsl/kodi-ac4). The installer
is on its [Releases](https://github.com/ferrellsl/kodi-ac4/releases) page.
The port is a single patch to FFmpeg 6.0.1,
[`0006-ffmpeg-ac4-decoder.patch`](https://github.com/ferrellsl/kodi-ac4/blob/main/patches/0006-ffmpeg-ac4-decoder.patch);
the FFmpeg 6.0.1 tree it comes from is the branch
[`ffmpeg-6.0.1`](https://github.com/ferrellsl/ffmpeg-ac4/tree/ffmpeg-6.0.1)
of this repository. Both decode the same (see "Changes in this repository"
below).

## What it decodes

- Mono, stereo and 5.1 AC-4 (the forms broadcasters mostly use), and 7.1.
- 5.1 with parametric surrounds (A-CPL) and the high-frequency extension
  (A-SPX), as ATSC 3.0 broadcasters send it.
- Several presentations in one stream: the decoder option `-presentation N`
  chooses one (0 is the first).

**Output sample rate.** AC-4 runs at an internal frame length tied to the
video frame rate, and the specification has the decoder resample it to the
external rate. This decoder outputs the internal rate and leaves the
resampling to the caller (FFmpeg's resampler, or Kodi's): **46034 Hz** for
29.97 fps (ATSC 3.0), 46080 Hz for 30 fps, 51200 Hz for 25 fps, 48000 Hz where
no resampling is needed. To get 48 kHz from the command line, add
`-ar 48000`.

## How it was checked

Against Dolby's own decode of the same program: the AC-4 Online Delivery
Kit 1.5 from Dolby carries the same material as AC-4 5.1 and as Dolby Digital
Plus 5.1 (decoded by FFmpeg's E-AC-3 decoder). Correlation of the decoded
waveforms below 4 kHz: center 1.00, LFE 0.96, front left/right 0.92/0.91,
the A-CPL surrounds 0.70/0.64; the channel levels agree within about half a
dB of each other (all about 4 dB below the DD+ track: its loudness
normalization differs). The kit's channel-identification signal decodes
with every channel within 1-2 dB of the DD+ version, second by second, and
its 29.97 fps version lasts as long as the reference.

## Known limitations

- **A-CPL is done for 5.x codec modes ASPX_ACPL_1 and ASPX_ACPL_2** (those
  found in ATSC 3.0 broadcasts and Dolby's test material). ASPX_ACPL_3 and
  A-CPL in 7.x elements are not done yet: no test material uses them so far.
- **Immersive AC-4 (IMS, A-JOC / Atmos, and 5.1.4 and other channel
  configurations with height channels) is not supported**: such frames are
  rejected as invalid data.

## Changes in this repository

- **Parametric surrounds (A-CPL) for 5.x.** Much 5.1 on ATSC 3.0 is sent as
  left, right, center and LFE, the surrounds described by parameters for the
  decoder to rebuild. The parameters were parsed but never used, so the
  surrounds were silent - and the fronts came out about 6 dB too quiet, since
  what is sent for them is a scaled mix of front and surround. Now done as in
  ETSI TS 103 190-1 V1.4.1 clause 5.7.7: differential decoding and
  dequantization of the alpha and beta parameters, the decorrelators and
  transient ducker, and the A-CPL modules in the QMF domain.
- **High frequencies (A-SPX) for 5.x.** A-SPX was applied to mono and stereo
  only, so 5.1 stopped at the core coder's bandwidth (about 14 kHz). The
  stereo and mono A-SPX processing now serves 5.x channels too, before A-CPL.
  (Decoder option `aspx_5x` turns it off, for comparison.)
- **Sample rate (speed and pitch).** The decoder output each frame at 48 kHz
  without the resampling the specification requires (TS 103 190-1 4.3.3.2.6,
  6.2.15): ATSC 3.0 audio (29.97 fps) played 4 % fast and 0.7 semitone sharp,
  25 fps material 6.7 % slow. It now reports the internal rate (see "Output
  sample rate" above).
- **Polarity.** Every channel came out inverted against Dolby's decode; the
  output is now the right way up.
- **Channel order (the LFE).** The decoder put the LFE after the main
  channels, where FFmpeg's 5.1 and 7.1 layouts expect it fourth
  (front left, front right, center, LFE, ...). In 5.1 the subwoofer got the
  left surround and the LFE came out of the right surround speaker. The
  output is now reordered to FFmpeg's layouts (see the end of
  `ac4_decode_frame` in `libavcodec/ac4dec.c`). Checked against 5.1 AC-4
  recorded from ATSC 3.0 broadcasts: the LFE is now output channel 4.
- **Immersive streams no longer crash the decoder.** In immersive (IMS)
  streams, channels could reach the spectrum reordering with window offsets
  left over from an earlier frame, and the decoder wrote far past its buffers
  (a segfault, which in Kodi closes the player). Each channel's window layout
  is now checked against the buffers first (`channel_windows_valid()`), and a
  frame that doesn't fit is rejected as invalid data.
- **No phantom AC-4 track from subtitles.** An MPEG-TS private stream that
  isn't otherwise identified is assumed to be AC-4, which also caught the
  STPP (TTML subtitle) stream of ATSC 3.0 broadcasts and listed it as a
  0-channel AC-4 track. A registration descriptor now decides: `AC-4` marks
  AC-4, anything else (such as `STPP`) leaves the stream as data.
- **Out-of-bounds reads and writes in window grouping.** The LFE channel looked
  up its window-grouping bit count with a transform index of 4, past the end
  of the tables; it is one long window, so it now uses none. The
  scale-factor grouping array also had room for one entry fewer than the
  decoder can write.
- **Building with current compilers.** `libavcodec/x86/mathops.h` has
  FFmpeg's later fix for inline assembly that newer assemblers reject
  ("operand type mismatch for `shr`"), so the tree builds with today's
  MSYS2/MinGW-w64 GCC.

## Building

As any FFmpeg of its time: `./configure` (for example
`--enable-shared --disable-static`), then `make`. It has been built with
MSYS2/MinGW-w64 and with Visual Studio 2019 (`--toolchain=msvc`).

---

FFmpeg README
=============

FFmpeg is a collection of libraries and tools to process multimedia content
such as audio, video, subtitles and related metadata.

## Libraries

* `libavcodec` provides implementation of a wider range of codecs.
* `libavformat` implements streaming protocols, container formats and basic I/O access.
* `libavutil` includes hashers, decompressors and miscellaneous utility functions.
* `libavfilter` provides a mean to alter decoded Audio and Video through chain of filters.
* `libavdevice` provides an abstraction to access capture and playback devices.
* `libswresample` implements audio mixing and resampling routines.
* `libswscale` implements color conversion and scaling routines.

## Tools

* [ffmpeg](https://ffmpeg.org/ffmpeg.html) is a command line toolbox to
  manipulate, convert and stream multimedia content.
* [ffplay](https://ffmpeg.org/ffplay.html) is a minimalistic multimedia player.
* [ffprobe](https://ffmpeg.org/ffprobe.html) is a simple analysis tool to inspect
  multimedia content.
* Additional small tools such as `aviocat`, `ismindex` and `qt-faststart`.

## Documentation

The offline documentation is available in the **doc/** directory.

The online documentation is available in the main [website](https://ffmpeg.org)
and in the [wiki](https://trac.ffmpeg.org).

### Examples

Coding examples are available in the **doc/examples** directory.

## License

FFmpeg codebase is mainly LGPL-licensed with optional components licensed under
GPL. Please refer to the LICENSE file for detailed information.

## Contributing

Patches should be submitted to the ffmpeg-devel mailing list using
`git format-patch` or `git send-email`. Github pull requests should be
avoided because they are not part of our review process and will be ignored.
