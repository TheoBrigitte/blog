---
title: "Efficient Video Transcoding for Home Media Servers"
date: 2025-10-01T23:00:00+02:00
description: "Automate video transcoding to H.264 for seamless streaming on Jellyfin, Plex, and other media servers"
---

Running a home media server on low-power hardware? Transcoding videos on-the-fly can turn your quiet server into a space heater. This script pre-transcodes videos to H.264 + AAC, ensuring smooth playback without real-time conversion.

## Why Pre-Transcode?

Real-time transcoding is CPU-intensive and wasteful when streaming content multiple times. By converting videos once to a widely compatible format, you get:

- **Lower CPU usage** - No on-the-fly transcoding needed
- **Reduced heat and noise** - Especially important for always-on boxes
- **Better compatibility** - H.264/AAC works everywhere
- **Energy efficiency** - Less power consumption, better for the planet

## The Script

This bash script uses FFmpeg in Docker to convert videos to an optimized format for streaming:

```bash
#!/bin/bash
#
# Author: Théo Brigitte
# Date: 2025-09-16
#
# Usage: transcode.sh input_file output_file
#
# Options:
#  --help                  Show this help message and exit
#  --[no-]audio     [lang]
#  --[no-]video     [lang]
#  --[no-]subtitles [lang] Exclude or include (default) the stream in the output
#                          file with optional optional language filter (e.g. "eng")
#                          see https://trac.ffmpeg.org/wiki/Map#Specificlanguage
#
#  --crf <value>           Set the Constant Rate Factor (CRF) for video quality
#                          (default: 23, lower is better quality, range 0-51)
#                          see https://trac.ffmpeg.org/wiki/Encode/H.264#crf
#
#  --tune <preset>         Set the tune preset for x264 encoder
#
#
# FFmpeg script to convert video files to h264 + aac format using Docker
# for better compatibility with streaming service like Jellyfin, Plex, etc.

set -euo pipefail

exit_error() {
  echo "[ERROR] $1"
  exit 1
}

print_help() {
  sed -ne '/Usage/,/^$/{p; /^$/q}' "$0" |sed -e '/^$/d; s/#\s\?//'
}

# Default options
video="0:v:0"     # keep first video
audio="0:a"       # keep all audio
subtitles="0:s"   # keep all subtitles
crf=23            # default CRF value
tune=""           # no tune by default

# Parse command line arguments
shopt -s extglob
while [[ $# -gt 0 ]]; do
  case $1 in
    -h|--help)
      print_help
      exit;;
    --?(no-)@(audio|video|subtitles))
      option="${1##*-}"
      test ! "${!option+set}" && exit_error "Invalid option $option"
      if [[ "$1" == --no-* ]]; then
        eval "${option}=-0:${option:0:1}"
      else
        filter=""
        if [[ -n "${2-}" && ! "$2" =~ ^- ]]; then
          filter=":m:language:${2}"; shift
        fi
        eval "${option}=0:${option:0:1}${filter}"
      fi;;
    --crf)
      test -z "${2-}" && exit_error "$1 requires an argument"
      crf="$2"; shift;;
    --tune)
      test -z "${2-}" && exit_error "$1 requires an argument"
      tune="-tune $2"; shift;;
    *)
      break;;
  esac
  shift
done

# Print help if wrong number of arguments
if [ "$#" -ne 2 ]; then
  print_help
  exit 1
fi

# Check arguments
input_file="$(readlink -e "$1")" || exit_error "File $1 does not exist"
input_dir="$(dirname "$input_file")" || exit_error "Directory for $input_file does not exist"
input_filename="$(basename "$input_file")" || exit_error "File $input_file does not exist"
output_file="$(readlink -f "$2")" || exit_error "File $2 does not exist"
output_file="${output_file%.*}.mp4"
output_dir="$(dirname "$output_file")" || exit_error "Directory for $output_file does not exist"
output_filename="$(basename "$output_file")" || exit_error "File $output_file does not exist"

# Create log directory
mkdir -p "$output_dir/log"
timestamp=$(date +%Y%m%d-%H%M%S)
log_file="log/ffmpeg-$timestamp-${output_filename%.*}.log"

echo "[INFO] Converting $input_filename" | tee -a "$output_dir/$log_file"

# Run ffmpeg in Docker
docker run --rm -it \
  -v "$input_dir:/input" \
  -v "$output_dir:/output" \
  -e FFREPORT="file=/output/${log_file}:level=32" \
  linuxserver/ffmpeg \
  -xerror \
  -i "/input/$input_filename" \
  -map "$video" \
  -map "$audio" \
  -map "$subtitles" \
  -movflags +faststart \
  -preset slow \
  -codec:v libx264 \
  -crf "$crf" \
  -maxrate 8259125 \
  -bufsize 16518250 \
  -profile:v high \
  -x264opts subme=0:me_range=16:rc_lookahead=10:me=hex:open_gop=0 \
  -force_key_frames "expr:gte(t,n_forced*3)" \
  -sc_threshold:v 0 \
  $tune \
  -vf format=yuv420p \
  -codec:a libfdk_aac \
  -ac 2 \
  -b:a 192k \
  -f mp4 \
  -y \
  "/output/$output_filename"

echo "[SUCCESS] Converted to $output_file" | tee -a "$output_dir/$log_file"
```

Save this as `transcode.sh` in your `~/.local/bin/` directory and make it executable:

```bash
chmod +x ~/.local/bin/transcode.sh
```

## How to Use

Basic usage:

```bash
transcode.sh input.mkv output.mp4
```

With options:

```bash
# Keep only French audio
transcode.sh --audio fre input.mkv output.mp4

# Remove all subtitles
transcode.sh --no-subtitles input.mkv output.mp4

# Higher quality (lower CRF)
transcode.sh --crf 20 input.mkv output.mp4

# Optimize for film
transcode.sh --tune film input.mkv output.mp4
```

## Key Features

**Stream Selection**: Filter by language or exclude entire stream types (audio/video/subtitles) using `--audio fre`, `--no-subtitles`, etc.

**Quality Control**: The `--crf` option controls quality (default 23, lower = better quality but larger files).

**Docker-Based**: Uses [linuxserver/ffmpeg](https://docs.linuxserver.io/images/docker-ffmpeg/) with libfdk_aac for high-quality AAC encoding.

**Streaming Optimized**:
- Fast start (`-movflags +faststart`) - playback begins before full download
- Keyframes every 3 seconds - efficient seeking
- H.264 high profile with optimal x264 settings

**Logging**: Creates timestamped logs in the output directory for debugging.

## Technical Details

The encoding settings are optimized for compatibility and efficiency:

- **Video**: H.264 (libx264) with CRF 23, high profile
- **Audio**: AAC (libfdk_aac) stereo at 192k
- **Container**: MP4 with faststart flag
- **Pixel format**: YUV420p for maximum compatibility

The x264 options (`subme=0:me_range=16:rc_lookahead=10:me=hex:open_gop=0`) balance encoding speed with quality, while forced keyframes ensure smooth seeking.

## Further Reading

- [FFmpeg H.264 Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/H.264)
- [FFmpeg AAC Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/AAC)
- [Jellyfin Codec Support](https://jellyfin.org/docs/general/clients/codec-support)
- [linuxserver/ffmpeg Docker Image](https://docs.linuxserver.io/images/docker-ffmpeg/)

---

**Installation**: Save the script to `~/.local/bin/transcode.sh` and run `chmod +x ~/.local/bin/transcode.sh`. Requires Docker.
