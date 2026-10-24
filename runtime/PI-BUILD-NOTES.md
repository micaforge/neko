# Pi-build notes — reference deltas for small ARM hosts

Deployment-level deltas for running neko on a Pi 4-class
host. No source changes; everything here is image content
or compose configuration.

## 1. Screen table trim

`runtime/xorg.conf` in this branch — see the PR body.
The stock-image equivalent is a bind mount of a trimmed
`xorg.conf` over `/etc/neko/xorg.conf`.

## 2. Hardware H.264 encode (V4L2)

The stock image already ships the GStreamer V4L2 plugins,
and neko accepts a custom video pipeline — so this is
compose-only:

- Pass the encoder device into the container:
  `devices: ["/dev/video11:/dev/video11"]`
  (`/dev/video11` is the bcm2835 codec encoder node).
- Pin the codec to match the pipeline's output:
  `NEKO_CAPTURE_VIDEO_CODEC: "h264"`
- Supply the pipeline (the `%s` is neko's display
  placeholder):
  `NEKO_CAPTURE_VIDEO_PIPELINE: "ximagesrc display-name=%s show-pointer=true use-damage=false ! video/x-raw,framerate=20/1 ! videoconvert ! queue ! video/x-raw,format=NV12 ! v4l2h264enc ! h264parse config-interval=-1 ! video/x-h264,stream-format=byte-stream,profile=constrained-baseline"`

Measured on the reference host, streaming to a connected
viewer at a 1024x780@20 desktop: container CPU **~216%
(default software VP8) → ~24%** sustained.

Notes:
- Setting a single custom pipeline collapses neko's
  multi-pipeline defaults to one stream definition; a
  one-off client negotiation warning at first connect
  was observed and did not recur.
- The switch is also a codec switch (VP8 → H.264) for
  every client — all current browsers accept H.264 in
  WebRTC.
- Encoder bitrate controls (`extra-controls`) are
  deliberately left at element defaults in this
  reference.
