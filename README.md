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
[`0006-ffmpeg-ac4-decoder.patch`](https://github.com/ferrellsl/kodi-ac4/blob/main/patches/0006-ffmpeg-ac4-decoder.patch),
and contains the channel-order fix below. It also has fixes that this 4.3 tree
does not:

- immersive (IMS) streams are rejected as invalid data instead of crashing the
  decoder;
- an ATSC 3.0 subtitle stream (STPP) is no longer taken for a 0-channel AC-4
  track;
- the LFE no longer reads past the window-grouping tables.

## What it decodes

- Mono, stereo and 5.1 AC-4 (the forms broadcasters mostly use), and 7.1.
- Several presentations in one stream: the decoder option `-presentation N`
  chooses one (0 is the first).

## Known limitations

- **Surround channels coded parametrically (A-CPL) are not rebuilt.** Much
  5.1 on ATSC 3.0 is sent as left, right, center and LFE, with the
  surrounds described by parameters for the receiver to reconstruct. This
  decoder reads those parameters but does not reconstruct the surrounds, so
  in such streams the two surround channels are silent.
- **A-SPX (the high-frequency extension) is applied only to mono and
  stereo.** 5.1 streams that use it decode with a reduced top end.
- **Immersive AC-4 (IMS, A-JOC / Atmos) is not supported**, and such streams
  can make the decoder fail.

## Changes in this repository

- **Channel order (the LFE).** The decoder put the LFE after the main
  channels, where FFmpeg's 5.1 and 7.1 layouts expect it fourth
  (front left, front right, center, LFE, ...). In 5.1 the subwoofer got the
  left surround and the LFE came out of the right surround speaker. The
  output is now reordered to FFmpeg's layouts (see the end of
  `ac4_decode_frame` in `libavcodec/ac4dec.c`). Checked against 5.1 AC-4
  recorded from ATSC 3.0 broadcasts: the LFE is now output channel 4.
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
