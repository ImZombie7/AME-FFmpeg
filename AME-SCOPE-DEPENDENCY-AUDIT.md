# AME-FFMPEG v0.1 — Initial Build and Dependency Audit

**Audit status:** In progress  
**Source reviewed:** FFmpeg fork, `master` branch  
**Scope authority:** AME-FFMPEG v0.1 scope agreed in project discussion  
**Important:** This is a source-backed build-system and component audit, not a line-by-line review of all 11,281 tracked files. No upstream source files are modified by this document.

## 1. Executive findings

FFmpeg already provides component-level build selection through its `configure` script. AME should initially preserve the upstream source tree and create a narrow build configuration. Do not delete source directories merely because their features are outside the first release.

The build can be narrowed at several levels:

- Libraries: `avutil`, `avcodec`, `avformat`, and selected supporting libraries.
- Programs: `ffmpeg` and optionally `ffprobe`.
- Encoders: native AAC, H.264 QSV, H.264 NVENC, and an explicitly chosen software H.264 fallback if required.
- Muxers/demuxers: MP4/MOV infrastructure and the input formats AME actually accepts.
- Protocols: local file/pipe I/O plus only the network transports AME chooses to support.
- Filters: only those needed for the first release's scaling, frame-rate and pixel-format operations.

## 2. Codec findings

### AAC

- FFmpeg has a native `aac` encoder.
- The encoder is registered as `ff_aac_encoder`.
- Its configure dependency selects `audio_frame_queue`, `lpc`, and `sinewin`.
- A separate external AAC library is not required for the native AAC encoder.

**Initial recommendation:** use native `aac` for v0.1. Keep other AAC implementations disabled unless a measured quality or compatibility requirement justifies them.

### H.264

The H.264 situation needs an explicit decision:

- FFmpeg includes a native H.264 decoder (`h264`).
- The inspected codec registration has hardware H.264 encoders for QSV and NVENC.
- The software H.264 encoders are external integrations, not a native general-purpose FFmpeg software H.264 encoder:
  - `libx264` requires the external x264 library and enables GPL-licensed code.
  - `libopenh264` requires the external OpenH264 library.

**Decision still needed:** whether AME v0.1 requires a software H.264 fallback. If yes, choose x264 or OpenH264 deliberately. Do not silently treat `h264` as a software encoder name; in FFmpeg it identifies the decoder.

### Intel Quick Sync Video

- H.264 QSV encoder registration is `h264_qsv`.
- Configure selects the shared `qsvenc` support for this encoder.
- QSV's configure dependency is represented through `libmfx`; the current configure script also supports Intel oneVPL through `--enable-libvpl`, with a minimum version check.
- The source contains QSV encoder/decoder support and QSV video-processing filters.

**Build requirement:** the build environment needs the matching Intel Media SDK / oneVPL development headers and libraries. Runtime operation also depends on a compatible Intel graphics driver and hardware. A successful build does not guarantee that a given PC can use QSV.

### NVIDIA NVENC

- H.264 NVENC encoder registration is `h264_nvenc`.
- Its configure dependency is `nvenc`, which depends on `ffnvcodec`.
- The configure script checks for the NVIDIA Video Codec SDK headers through the `ffnvcodec` pkg-config package.
- The NVENC integration dynamically loads the NVIDIA runtime API; the build does not imply that every machine has a compatible NVIDIA GPU or driver.

**Build requirement:** provide the required ffnvcodec headers/package to the build. Runtime availability must be detected separately.

## 3. MP4 container findings

- `mp4` is registered as an output muxer.
- Configure declares `mp4_muxer_select="mov_muxer"`.
- The shared MOV muxer implementation is built from `libavformat/movenc.c` and related objects.
- The MOV muxer selects supporting ISO media, ISO writer, RIFF, RTP-chain and bitstream-filter components, including AAC-to-ADTS conversion support.

**Implication:** do not try to isolate MP4 by deleting MOV-related source. The MP4 muxer relies on shared MOV/ISO media infrastructure.

The first release should separately decide which input demuxers are needed. An MP4 output target alone does not define the input media formats AME must accept.

## 4. Scaling and audio support

- `libswscale` is FFmpeg's image scaling and pixel-format conversion library.
- The `scale` filter is built from `vf_scale.c` and supporting code.
- QSV has a separate `scale_qsv` / VPP filter path, dependent on QSV support.
- `libswresample` provides audio sample-format, sample-rate and channel-layout conversion.
- `libavfilter` is a separate library; it should only be included if AME uses filter graphs or filter-based processing.

**Initial recommendation:** retain `swscale` and `swresample`. Decide whether AME's first implementation uses `libavfilter` or performs a narrow set of conversions directly through the library APIs.

## 5. Streaming and I/O

FFmpeg separates protocols from containers and codecs. The inspected format Makefile has independently configurable protocol objects, including:

- Local I/O: `file`, `pipe`
- Network: `tcp`, `udp`
- Higher-level transports: `rtp`, `rtmp`, `http`, and others

TCP and UDP protocol configuration selects the network support component. RTP selects UDP support.

The scope phrase “network transport” is not specific enough to justify enabling every protocol. AME should choose transport(s) based on its actual stream design. For the Encode server/client idea, a custom framed stream over TCP may be a better-defined first transport than enabling the entire FFmpeg protocol catalogue.

For an in-process API, FFmpeg also supports custom I/O through its AVIO interfaces. That is distinct from enabling a network protocol.

## 6. Libraries, CLI and metadata

- `avutil` provides shared data structures and utility functions.
- `avcodec` provides codec contexts, packets, frames and codec APIs.
- `avformat` provides container demuxing/muxing, stream metadata and AVIO.
- `ffmpeg` is a separately configured CLI program.
- `ffprobe` is a separately configured inspection CLI program.
- The configure script includes examples demonstrating metadata access and codec/container operations.

**Implication:** AME can use the libraries directly and need not shell out to the FFmpeg executable forever. The CLI can remain useful for diagnostics, compatibility testing and developer workflows.

## 7. Build-system behavior verified

The current `configure` script exposes:

- `--disable-everything` to clear the component lists.
- Individual `--enable-encoder`, `--enable-decoder`, `--enable-muxer`, `--enable-demuxer`, `--enable-protocol`, and `--enable-filter` switches.
- Library/program controls such as `--disable-avdevice`, `--disable-avfilter`, `--disable-ffplay`, and `--disable-doc`.
- Hardware integration controls for ffnvcodec, libmfx and libvpl.
- Dependency expansion and ordering in the configure script, which resolves selected component dependencies before generating the component registration lists and build configuration.

One subtlety: `--disable-all` is not identical to `--disable-everything`. In the inspected script, `--disable-all` also disables libraries/programs/docs and then re-enables `avutil`. Treat these switches carefully when constructing the final build.

## 8. Preliminary component disposition

| Component | v0.1 disposition | Reason |
|---|---|---|
| libavutil | KEEP | Core frame, packet, memory, option and utility infrastructure |
| libavcodec | KEEP | AAC/H.264 codec APIs and implementations |
| libavformat | KEEP | MP4 muxing, demuxing, metadata and AVIO |
| libswscale | KEEP / likely required | Scaling and pixel-format conversion |
| libswresample | KEEP / likely required | Audio conversion/resampling |
| libavfilter | OPTIONAL | Keep if filter-graph processing is part of v0.1 |
| libavdevice | EXCLUDE FROM INITIAL BUILD unless needed | Device capture/output is not in the agreed v0.1 scope |
| ffmpeg CLI | KEEP | CLI is explicitly in scope |
| ffprobe CLI | KEEP / useful | Metadata inspection and diagnostics |
| ffplay | EXCLUDE FROM INITIAL BUILD | Playback is not in scope |
| Tests and FATE data | KEEP IN SOURCE | Exclude from runtime package, not from the source fork |
| docs and examples | KEEP IN SOURCE | Useful for development and API understanding |
| unrelated codec implementations | EXCLUDE FROM BUILD | Prefer configure selection over source deletion |

## 9. Open decisions before writing the production configure preset

1. **Software H.264 fallback:** x264 (GPL) or OpenH264 (external integration), or no software fallback in v0.1.
2. **Input formats:** which formats must Encode hand to AME, beyond the current Premiere frame-server/signpost workflow.
3. **Network transport:** TCP stream, UDP/RTP, or another explicitly selected protocol.
4. **Filter architecture:** FFmpeg filter graphs versus direct library conversion for the initial feature set.
5. **Windows build target:** compiler/toolchain and how Intel oneVPL and NVIDIA ffnvcodec development dependencies are supplied in CI.
6. **Packaging model:** static or shared libraries, and how runtime driver/API dependencies are discovered.

## 10. Source files inspected in this pass

- `configure`
- `libavcodec/Makefile`
- `libavcodec/allcodecs.c`
- `libavformat/Makefile`
- `libavformat/allformats.c`
- `libavutil/Makefile`
- `libavfilter/Makefile`
- `libswscale/Makefile`
- `libswresample/Makefile`
- `fftools/Makefile`

This list is the first dependency-focused review set, not the complete repository audit. Subsequent passes should inspect the implementation files, public API headers, Windows platform code, hardware backends, test coverage, and licensing of every enabled component.
