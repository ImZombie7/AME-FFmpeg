# AME-FFMPEG — Library API and Frame Pipeline Notes

**Audit status:** In progress  
**Purpose:** Map the agreed frame/API and streaming scope onto FFmpeg's existing public interfaces.

## 1. Core encode pipeline

FFmpeg's public API already exposes the required frame-to-packet pipeline:

1. Allocate and configure an `AVCodecContext` for the selected encoder.
2. Open it with `avcodec_open2()`.
3. Supply uncompressed video/audio as `AVFrame` objects through `avcodec_send_frame()`.
4. Receive encoded data as `AVPacket` objects through `avcodec_receive_packet()`.
5. Configure an output `AVFormatContext` and output stream(s).
6. Write the container header with `avformat_write_header()`.
7. Write encoded packets with `av_interleaved_write_frame()` (or the lower-level frame write API where appropriate).
8. Flush the encoder and write the trailer before closing resources.

This is a suitable foundation for a library/API. AME does not need to invent its own raw-frame, packet, timestamp or codec-context abstractions before it can encode.

## 2. Proposed AME boundary

Keep FFmpeg types inside the engine boundary initially, but expose an AME-owned API to Encode:

- **AME session/configuration:** input dimensions, pixel format, frame rate, audio layout, target container and output destination.
- **Encoder selection:** software H.264, QSV H.264, NVENC H.264, and native AAC, subject to the v0.1 decisions.
- **Frame submission:** submit video/audio frames with timestamps and explicit ownership/lifetime rules.
- **Packet/mux stage:** encoded packets are sent to the selected muxer.
- **Metadata query:** source stream information, duration, dimensions, frame rate, bitrate and stream metadata.
- **Progress/events:** encoded frame/packet counts, output bytes, errors and completion.
- **Cancellation:** stop accepting frames, flush or abort according to a documented cancellation policy, then release contexts safely.

Do not expose every FFmpeg option directly as the permanent public AME API. Keep an internal FFmpeg adapter so the UI's Premiere-style controls can map to encoder-specific options without leaking backend flags into the GUI.

## 3. Frame input/output and AVIO

The public `AVIOContext` API supports caller-provided I/O callbacks through `avio_alloc_context()`. This allows custom input/output sources without requiring AME to use a built-in URL protocol for every transport.

Important distinction:

- **Frame API:** `AVFrame` in, `AVPacket` out. This is the codec-level interface.
- **Container I/O:** `AVFormatContext` plus `AVIOContext`. This writes or reads a media container.
- **Network transport:** socket or transport implementation carrying bytes/packets/frames. It is not automatically the same thing as a codec frame API.

For AME ServerMode/ClientMode, define the wire protocol separately. A raw sequence of AVFrame memory buffers is not a safe network protocol: pixel format, dimensions, plane strides, timestamps, packet boundaries, versioning, backpressure and ownership all need explicit handling. A framed message protocol or encoded packet stream is a better boundary.

## 4. Hardware encoder integration findings

### QSV

The inspected QSV encoder implementation:

- Initializes an Intel Media SDK / oneVPL session.
- Maps FFmpeg frames and pixel formats to QSV surfaces.
- Submits frames to the hardware encoder and returns packets.
- Has explicit hardware-frame and device-context configuration paths.

The QSV implementation is not just a different codec string. It has a hardware session, surface formats, device setup and frame-transfer behavior. AME should model hardware selection and input pixel-format compatibility explicitly.

### NVENC

The inspected NVENC implementation:

- Dynamically loads the NVIDIA encoding runtime libraries.
- Opens an NVENC encode session.
- Checks codec/device capabilities.
- Sets rate-control and H.264 codec configuration.
- Allocates and manages input surfaces, then receives encoded packets.

NVENC availability is therefore a runtime capability, not something that can be inferred from the fact that the application was built with NVENC support.

## 5. Metadata

The public format API centers on `AVFormatContext` and `AVStream`. FFmpeg's API documentation describes stream arrays, stream parameters, metadata dictionaries, duration/timing and custom I/O.

For source inspection, the engine should use the demuxer/probe path to discover stream data rather than guess based on filename extensions. For a first metadata API, report at least:

- Container/format name
- Duration when known
- Stream count and stream types
- Video width/height
- Video frame rate and time base (keep these distinct)
- Audio sample rate and channel layout
- Bitrate when reported or reliably calculable
- Codec name/profile/pixel or sample format
- Global and per-stream metadata

Some files do not contain a reliable duration or bitrate. The API must represent unknown values instead of fabricating numbers.

## 6. Lifecycle and error-handling implications

The API should make these rules explicit:

- Every allocated FFmpeg object has one clear owner and release path.
- Encoder send/receive loops must handle backpressure and `EAGAIN`; one call is not guaranteed to produce a packet.
- End-of-stream requires flushing/draining the encoder.
- Packet timestamps must be rescaled to the output stream time base before muxing when the time bases differ.
- Muxer header/trailer ordering is mandatory.
- Cancellation and failure paths must close partially initialized contexts without double-freeing resources.
- Preserve FFmpeg's detailed error codes and log messages in AME diagnostics. A UI should show the useful error text, not only an unsigned process exit code.

## 7. Source files inspected in this pass

- `libavcodec/qsvenc.c`
- `libavcodec/qsv_api.c`
- `libavcodec/nvenc.c`
- `libavcodec/nvenc_h264.c`
- `libavcodec/aacenc.c`
- `libavformat/movenc.c`
- `libavcodec/avcodec.h`
- `libavformat/avformat.h`
- `libavformat/avio.h`
- `libavutil/hwcontext_qsv.h`

This is a focused API and hardware-path review, not a complete audit of each file or the whole repository.
