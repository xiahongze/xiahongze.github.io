---
layout: post
title: Decoding Audio With ffmpeg
date:   2019-11-22 22:00:00 +1100
categories: tech
comments: true
---

Recently, I got a chance to deal with [ffmpeg](https://www.ffmpeg.org) at library
level. It is a great library containing everything one needs to work with audio
and video encoding/decoding. Using the command line utils is already quite
powerful. Not only can it convert between all sorts of formats, it can also
stream from remote addresses. Fantastic! However, sometimes one wants to call
the library without any proxy. This becomes a medium to hard level task,
especially when there are not many up to date examples online.

Firstly, I would like to introduce a few concepts before showing the actual code.

## Concepts

- `AVFormatContext`: holds reference to the file being opened. It has to be allocated
    and then intialized with `avformat_open_input`. If a customized IO is preferred
    over a conventional file IO, then one could provide an `AVIOContext` as to
    `format->pb`. Please note that `AVIOContext` needs to be allocated with
    `avio_alloc_context`.
- `AVCodec`: the codec for encoding and decoding. Each stream in a file might have
    a different codec which can be retrieved from `stream->codecpar`, the codec
    parameters. Inside `codecpar`, `codec_type` and `codec_id` is available.
    `ffmpeg` supports many kinds of codec.
- `AVCodecContext`: holds a workspace for the encoding/decoding work for a given
    `codec`. It has to be allocated with `avcodec_alloc_context3` and initialized
    `avcodec_parameters_to_context` with `codecpar`.
- `AVPacket`: one frame of data extracted from `AVFormatContext`. This not only
     contains audio but also video from the origin file. To decode the content,
     firstly, supply `avcodec_send_packet` with the packet and the corresponding
     `codec_ctx`. Then, pass the decoded content in `AVCodecContext` to an
     `AVFrame` with `avcodec_receive_frame`.
- `AVFrame`: contains decoded data and parameters like channels, colors, packet
    size, nb\_frames, etc.

## High Level Workflow

- Init `AVFormatContext` and open file with it.
- Find the stream of your interest. In the example, I will use the first audio stream.
- Extract `codecpar` from the stream and select and configure an `AVCodec` and
    create an `AVCodecContext` workspace.
- Loop through all frames in `AVFormatContext` with `av_read_frame`.
  - Decode packet with the `AVCodecContext`
  - Extract data to an `AVFrame`
  - Process the `AVFrame`
- Close or unref all contexts, packets and frames

## Code

Finally, congratulations on getting to the last and fun bit of this post. In the
code example, I will show how one can extract numerical data from a wav file,
and sum them up to one number.

Here is the [link](https://github.com/xiahongze/ffmpeg-audio-decode-example) to the repository.

Please be mindful that I used a fixed data type (short) as I know that the wav
is encoded in such format. For other audio files, such as MP3, it might be
float or double depending on the encoder. To find out what format the data is,
simply look at `stream->codecpar->format` and compare with what is in `AVSampleFormat`.
It is possible to create some logic dealing with different format intelligently
but it would complicate this simple demo. Hence, one could try it out by oneself.

Another approach to solving multiple format problem is to use the resampler from
ffmpeg. Here is a [post](https://rodic.fr/blog/libavcodec-tutorial-decode-audio-file/)
talking about it, from which I learnt a lot. The improvement in my code is that
the usage of ffmpeg is updated and fixing some bugs and warnings. Calling the
resampler also implies extra overheads in extracting the audio data.