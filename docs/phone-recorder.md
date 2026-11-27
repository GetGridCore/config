# Phone Recording

## Tips for good recordings

### Record from the browser, not the home-screen app

Recording from the installed (home-screen / pinned) app performs noticeably worse than
recording from the browser directly - iOS throttles background drawing in app mode. Open the
dashboard in Safari itself when recording.

### Don't preview on the recording phone

Watching the live preview on the same phone that is recording pulls the video right back
down over the same WiFi link, roughly doubling the traffic. Preview from another device -
any dashboard can preview an ongoing phone recording, not just the one that started it.

### Phone condition matters

A hot, low-battery phone in direct sun throttles quickly. Low power mode hurts too. Keep the
phone charged and out of the sun where possible; expect degraded frame rates on hot days.

### File sizes can exceed the quality setting

Some phones (notably iPhones) encode at a higher data rate than requested, so files can be
noticeably larger than the nominal quality tier suggests - and the upload needs
proportionally more WiFi capacity.

### What the status labels mean

**DEGRADED** means the recording runs below its target frame rate. **NO UPLOAD** means video
is not reaching the server - the phone may still be recording and retrying, and brief
network dropouts heal on their own once the connection returns.

## Design decisions (internal)

### Frame rate: record at 25 fps

25 fps is the deliberate target, especially at 720p. In on-track testing a phone asked to
record at 30 fps sagged to ~18 fps and the upload fell behind real time; at 25 fps the same
phone paced its target exactly. While 30 fps will be achievable in favorable conditions
(recent phone, well charged, climate conditioned cabin), we choose to have more headroom at
the moment.

### The upload must keep up with the video

The recording only exists on the dashboard server - the phone uploads it live and keeps
nothing. This means the WiFi link has to sustain the video's data rate:

- If the link is slower than the video rate, the copy on the server falls behind real time.
  The recording is still fine and keeps uploading; it is just late.
- Brief network dropouts self-heal: the phone queues unsent video and catches up in order
  when the connection returns.
- If the backlog grows hopelessly large (server unreachable for minutes), the recording
  stops with a clear reason instead of silently eating phone memory.
