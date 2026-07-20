# gopus

Go bindings for the [Opus](https://opus-codec.org/) audio codec.

This module links against a prebuilt static Opus library (**Opus 1.6.1**) via
cgo, so you can encode and decode Opus audio directly from Go without shipping
any extra DLLs.

## Project layout

```
gopus/
├── go.mod
├── opus_nonshared.go     # bindings for windows/linux/etc. (amd64 & 386, static link)
├── opus_shared.go        # bindings using system Opus via pkg-config (other platforms)
└── opus/
    ├── include/          # Opus public headers (opus.h, opus_defines.h, ...)
    └── lib/libopus.a     # prebuilt static Opus library
```

The static library and headers under `opus/` are bundled with the module, so
`go build` works out of the box as long as you have a working C toolchain.

## Requirements

- Go with cgo enabled (`CGO_ENABLED=1`).
- A C compiler. On Windows use an mingw-based toolchain (e.g. `clang` /
  `gcc` from **llvm-mingw**).

> **Toolchain note:** on Windows, the C compiler used by cgo must belong to the
> same toolchain family that produced `opus/lib/libopus.a` (llvm-mingw).
> Mixing MSVC with a mingw `.a` will fail to link.

## Usage

Build/run with cgo enabled and the matching compiler selected:

```cmd
set CGO_ENABLED=1
set CC=clang
go run .
```

Minimal encode → decode example:

```go
package main

import (
	"fmt"

	gopus "github.com/kalinrin/gopus"
)

func main() {
	const (
		sampleRate = 48000 // Hz
		channels   = 1
		frameSize  = 960 // samples per channel (20 ms @ 48 kHz)
	)

	enc, err := gopus.NewEncoder(sampleRate, channels, gopus.Voip)
	if err != nil {
		panic(err)
	}
	dec, err := gopus.NewDecoder(sampleRate, channels)
	if err != nil {
		panic(err)
	}

	// A frame of PCM (int16) samples. Here it is just silence.
	pcm := make([]int16, frameSize*channels)

	// Encode one frame into an Opus packet (max 4000 bytes buffer).
	packet, err := enc.Encode(pcm, frameSize, 4000)
	if err != nil {
		panic(err)
	}

	// Decode the packet back into PCM.
	out, err := dec.Decode(packet, frameSize, false)
	if err != nil {
		panic(err)
	}

	fmt.Printf("encoded %d bytes, decoded %d samples\n", len(packet), len(out))
}
```

## API overview

**Encoder**

```go
enc, err := gopus.NewEncoder(sampleRate, channels int, app gopus.Application)
packet, err := enc.Encode(pcm []int16, frameSize, maxDataBytes int) ([]byte, error)

enc.SetBitrate(bitrate int)   // bits per second, or gopus.BitrateMaximum
enc.Bitrate() int
enc.SetVbr(vbr bool)
enc.SetApplication(app gopus.Application)
enc.Application() gopus.Application
enc.ResetState()
```

**Decoder**

```go
dec, err := gopus.NewDecoder(sampleRate, channels int)
pcm, err := dec.Decode(data []byte, frameSize int, fec bool) ([]int16, error)
dec.ResetState()
```

**Helpers & constants**

```go
n, err := gopus.CountFrames(data []byte) // number of Opus frames in a packet

// Application modes:
gopus.Voip
gopus.Audio
gopus.RestrictedLowDelay
```

`Encode`/`Decode` return one of the exported errors (`gopus.ErrBadArgument`,
`gopus.ErrInvalidPacket`, ...) when the underlying Opus call fails.

## License

The Go bindings are provided under this repository's license. Opus itself is
distributed under the BSD-3-Clause license by the Xiph.Org Foundation.
