# FFmpeg 6.0.1 with an AC-4 audio decoder

This branch is FFmpeg 6.0.1 - the FFmpeg that Kodi 21.3 "Omega" builds - with an
**AC-4 audio decoder**: Dolby's audio format used by ATSC 3.0 ("NextGen TV")
broadcasts, some DVB services and streaming. The decoder,
`libavcodec/ac4dec.c`, is Paul B Mahol's, ported here from the FFmpeg 4.3 tree
on this repository's `master` branch; AC-4 is also read from raw `.ac4` files,
MP4 and MPEG-TS.

It is the FFmpeg behind the Kodi 21.3 build with AC-4 at
[ferrellsl/kodi-ac4](https://github.com/ferrellsl/kodi-ac4) (installer on its
[Releases](https://github.com/ferrellsl/kodi-ac4/releases) page); the patch
there is `git diff` of this branch against its first commit (FFmpeg 6.0.1 as
released). `master` (FFmpeg 4.3) has the same decoder and decodes the same;
its README describes the decoder in full: what it decodes, how it was checked
against Dolby's own decode, its limitations, and every change made to it.

## In short

- Mono, stereo and 5.1 AC-4, including 5.1 with parametric surrounds (A-CPL,
  codec modes ASPX_ACPL_1/_2) and the high-frequency extension (A-SPX), as
  ATSC 3.0 broadcasters send it.
- **Output sample rate:** the decoder outputs AC-4's internal rate and leaves
  the resampling the specification requires to the caller - **46034 Hz** for
  29.97 fps (ATSC 3.0), 46080 Hz for 30 fps, 51200 Hz for 25 fps. From the
  command line, add `-ar 48000`.
- Not supported: immersive AC-4 (IMS / A-JOC, height channels such as 5.1.4),
  A-CPL codec mode ASPX_ACPL_3 and A-CPL in 7.x.
- Decoder options: `presentation` (which presentation, 0 is the first) and
  `aspx_5x` (A-SPX in 5.x, on by default; off for comparison).

## Commits on this branch

1. FFmpeg 6.0.1 as released (Kodi's tarball).
2. The decoder and demuxer ported to the 6.0 API, with the LFE channel order,
   MPEG-TS detection (ATSC 3.0 subtitles are not taken for AC-4) and the crash
   guard for immersive streams.
3. A-CPL for 5.x.
4. A-SPX for 5.x.
5. Output sample rate and polarity, checked against Dolby's DD+ decode of the
   same program.

---

FFmpeg README
=============

FFmpeg is a collection of libraries and tools to process multimedia content
such as audio, video, subtitles and related metadata.

## Libraries

* `libavcodec` provides implementation of a wider range of codecs.
* `libavformat` implements streaming protocols, container formats and basic I/O access.
* `libavutil` includes hashers, decompressors and miscellaneous utility functions.
* `libavfilter` provides means to alter decoded audio and video through a directed graph of connected filters.
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
