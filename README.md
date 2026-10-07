# Invotyx YT Stream SDK for Roku

Invotyx YT Stream SDK is a Roku SceneGraph `ComponentLibrary` that accepts a YouTube video ID and asynchronously returns a Roku-compatible playback URL.

The SDK is resolver-only. It does not create or control a Roku `Video` node, playback screen, seek bar, analytics, or advertisements. The host application owns the complete playback experience.

For supported on-demand videos and Shorts, the primary result is a locally served DASH manifest combining adaptive AVC video representations with AAC audio. The result can also contain a muxed MP4 fallback. Live broadcasts use the signed HLS or DASH manifest supplied by the source.

## Features

- A single `resolve(videoId)` API for regular videos, Shorts, live broadcasts, and live replays
- Adaptive DASH playback with separate high-quality video and audio tracks
- HLS or DASH playback for supported live streams
- Muxed MP4 fallback URL when available
- Observable asynchronous results and structured errors
- Token-gated initialization
- App-owned Roku `Video` node and playback UI
- No RapidAPI dependency or fallback inside the SDK

## Requirements

- Roku SceneGraph channel
- `rsg_version=1.3` in the host channel manifest
- Network access to the SDK package, Invotyx YT Stream service, YouTube, and resolved media hosts
- An SDK token issued by Invotyx Software Company

To request authorized access, contact [Invotyx](https://invotyx.com/).

## 1. Add the ComponentLibrary

Add the released SDK package to the main Scene's `children`.

```xml
<ComponentLibrary
    id="YTStreamLib"
    uri="https://github.com/RokuProducts/InvotyxYTStreamLib-Releases/releases/download/v1.0.0/InvotyxYTStreamLib_1.0.0.pkg"
/>
```

During local development, the SDK ZIP can be bundled inside the host channel:

```xml
<ComponentLibrary
    id="YTStreamLib"
    uri="pkg:/images/InvotyxYTStreamLib.zip"
/>
```

The filename and folder are examples; use the actual location included in the host channel package.

The host application's player remains separate:

```xml
<Video
    id="videoPlayer"
    width="1280"
    height="720"
    enableUI="true"
/>
```

## 2. Create and initialize the resolver

Wait until the ComponentLibrary reports `loadStatus = "ready"`. Create the namespaced resolver, append it to the Scene, register observers, and then call `initialize()`.

The resolver must remain attached for the complete playback session because it hosts generated DASH manifests locally on the Roku device.

```brightscript
sub init()
    m.videoPlayer = m.top.FindNode("videoPlayer")
    m.ytStreamLib = m.top.FindNode("YTStreamLib")

    m.ytStreamLib.ObserveField(
        "loadStatus",
        "OnYTStreamLibStatus"
    )
    m.videoPlayer.ObserveField("state", "OnVideoStateChanged")
end sub

sub OnYTStreamLibStatus()
    if m.ytStreamLib.loadStatus = "ready"
        resolver = CreateObject(
            "roSGNode",
            "InvotyxYTStreamLib:YTStreamResolver"
        )

        if resolver = invalid
            print "YT STREAM: Resolver component could not be created"
            return
        end if

        m.top.AppendChild(resolver)
        resolver.ObserveField("result", "OnYTStreamResult")
        resolver.ObserveField("error", "OnYTStreamError")

        authorized = resolver.CallFunc("initialize", {
            Token: "YOUR_INVOTYX_SDK_TOKEN"
        })

        if authorized
            m.ytStreamResolver = resolver
            print "YT STREAM: SDK ready, version "; resolver.version
        else
            m.top.RemoveChild(resolver)
            print "YT STREAM: SDK authorization failed"
        end if
    else if m.ytStreamLib.loadStatus = "failed"
        print "YT STREAM: ComponentLibrary failed to load"
    end if
end sub
```

The `Token` key is case-sensitive. Do not call `resolve()` unless initialization returns `true`.

## 3. Resolve a YouTube video ID

Pass the YouTube video ID only, not a complete YouTube URL:

```brightscript
videoId = "nrMXHTaVYgA"
m.requestedVideoId = videoId

accepted = m.ytStreamResolver.CallFunc("resolve", videoId)
if not accepted
    print "YT STREAM: Resolution request was rejected"
end if
```

Resolution is asynchronous. A `true` return value means the request was accepted; the playable URL is delivered later through the observable `result` field.

Use one resolver for the application and keep only one active resolution request at a time.

## 4. Play the returned stream

Ignore empty or stale notifications, verify the returned `videoId`, and assign both `result.url` and `result.streamFormat` to the application-owned player.

```brightscript
sub OnYTStreamResult()
    result = m.ytStreamResolver.result
    if result = invalid or result.Count() = 0 then return
    if result.videoId <> m.requestedVideoId then return

    if result.success <> true
        print "YT STREAM resolution failed: "; result.error
        ShowPlaybackError()
        return
    end if

    m.fallbackMuxedUrl = result.fallbackMuxedUrl
    m.usingMuxedFallback = false

    content = CreateObject("roSGNode", "ContentNode")
    content.url = result.url
    content.streamFormat = result.streamFormat

    m.videoPlayer.content = content
    m.videoPlayer.control = "play"
end sub
```

An on-demand DASH result has this shape:

```brightscript
{
    success: true
    videoId: "nrMXHTaVYgA"
    url: "http://127.0.0.1:8998/stream.mpd?videoId=..."
    dashUrl: "http://127.0.0.1:8998/stream.mpd?videoId=..."
    hlsUrl: ""
    fallbackMuxedUrl: "https://...googlevideo.com/videoplayback?..."
    streamFormat: "dash"
    quality: ""
    resolver: "..."
    error: ""
}
```

For a live result, `streamFormat` can be `hls` or `dash`, and `url` contains the source manifest URL.

Resolved media URLs are signed and expire. Do not save them in the registry or reuse them in a later session. Resolve the video ID again before playback.

## 5. Use the muxed fallback

The SDK returns `fallbackMuxedUrl` but does not start fallback playback. The host application should retry it once if the primary DASH stream enters the `error` state.

```brightscript
sub OnVideoStateChanged()
    if m.videoPlayer.state <> "error" then return
    if m.usingMuxedFallback then return
    if m.fallbackMuxedUrl = invalid or m.fallbackMuxedUrl = "" then return

    m.usingMuxedFallback = true

    fallbackContent = CreateObject("roSGNode", "ContentNode")
    fallbackContent.url = m.fallbackMuxedUrl
    fallbackContent.streamFormat = "mp4"

    m.videoPlayer.control = "stop"
    m.videoPlayer.content = fallbackContent
    m.videoPlayer.control = "play"
end sub
```

Not every video or live broadcast provides a muxed fallback. If `fallbackMuxedUrl` is empty, use the application's normal playback-error handling.

## Public API

### Functions

| Function | Input | Return value | Description |
| --- | --- | --- | --- |
| `initialize(params)` | `{ Token: String }` | Boolean | Authorizes and starts the resolver |
| `resolve(videoId)` | YouTube video ID | Boolean | Starts asynchronous resolution |

### Resolver fields

| Field | Type | Description |
| --- | --- | --- |
| `initialized` | Boolean | `true` after successful initialization |
| `sdkState` | String | Current SDK lifecycle state |
| `videoId` | String | Video ID associated with the current request |
| `result` | Associative array | Asynchronous playback result |
| `error` | Associative array | Structured initialization or resolution error |
| `version` | String | SDK version |

Implemented `sdkState` values:

| State | Meaning |
| --- | --- |
| `created` | Resolver node exists but is not initialized |
| `ready` | Initialization succeeded |
| `resolving` | Stream resolution is running |
| `preparing_dash` | Stream metadata is ready and the local DASH service is starting |
| `resolved` | A playable result was published |
| `error` | Initialization or resolution failed |

### Result fields

| Field | Type | Description |
| --- | --- | --- |
| `success` | Boolean | Whether resolution succeeded |
| `videoId` | String | Video ID associated with this result |
| `url` | String | Recommended primary playback URL |
| `dashUrl` | String | DASH URL when the primary result is DASH |
| `hlsUrl` | String | HLS URL when the primary result is HLS |
| `fallbackMuxedUrl` | String | Muxed MP4 fallback when available |
| `streamFormat` | String | `dash`, `hls`, or `mp4` |
| `quality` | String | Reserved quality description; may currently be empty |
| `resolver` | String | Internal resolver identifier |
| `error` | String | Empty on success; failure message otherwise |

### Error fields

```brightscript
sub OnYTStreamError()
    errorInfo = m.ytStreamResolver.error
    if errorInfo = invalid or errorInfo.Count() = 0 then return

    print "YT STREAM error code: "; errorInfo.code
    print "YT STREAM error message: "; errorInfo.message
    print "YT STREAM error stage: "; errorInfo.stage
    print "YT STREAM video ID: "; errorInfo.videoId
end sub
```

The error object contains `code`, `message`, `stage`, and `videoId`.

## Recommended lifecycle

1. Load one ComponentLibrary from the main Scene.
2. Wait for `loadStatus = "ready"`.
3. Create and append one `YTStreamResolver`.
4. Observe `result` and `error`.
5. Call `initialize()` once with the assigned token.
6. Call `resolve(videoId)` when playback is requested.
7. Keep the resolver attached while the returned stream is playing.
8. Let the host application manage the player, UI, analytics, ads, and muxed fallback.

## Troubleshooting

### ComponentLibrary fails to load

- Confirm that the package URL is accessible from the Roku device.
- For a bundled local ZIP, confirm the file is included in the host channel package and the `pkg:/` path matches exactly.
- Confirm the SDK package manifest declares `sg_component_libs_provided=InvotyxYTStreamLib`.
- Wait for `loadStatus = "ready"` before creating the resolver.

### Resolver is not ready

- Call `initialize()` after appending the resolver to the Scene.
- Use the exact `Token` field name.
- Confirm `initialized = true` and `sdkState = "ready"` before resolving a video.

### DASH fails immediately

- Keep the resolver attached and alive throughout playback.
- Assign both `ContentNode.url` and `ContentNode.streamFormat` from the result.
- Ensure another service is not using local TCP port `8998`.
- Retry `fallbackMuxedUrl` as `mp4` when it is available.

### Playback starts below 1080p

The generated DASH manifest can expose representations up to 1080p. Roku chooses the active representation according to bandwidth, buffer health, and device decoder capability, so quality may increase after playback stabilizes.

### A previously working URL stopped playing

Resolved media URLs expire. Call `resolve(videoId)` again instead of caching the playback URL.

## Distribution and security

The source repository is private. Approved applications consume the signed `.pkg` published through the public binary-release repository.

Do not publish SDK source, customer tokens, service credentials, signing keys, or internal implementation details. SDK access is licensed for authorized applications only. Reverse engineering, redistribution, modification, or repackaging is prohibited unless Invotyx provides written permission.

## License

Copyright © Invotyx Software Company. All rights reserved. See [LICENSE](LICENSE).
