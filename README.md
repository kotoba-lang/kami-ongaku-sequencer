# kami-ongaku-sequencer

Portable `.cljc` MIDI-equivalent event/pattern/track data model — the L3
sequencing/piano-roll layer of the `ongaku` (music production) engine stack
defined in [ADR-2607121400](https://github.com/com-junkawasaki/root/blob/main/90-docs/adr/2607121400-kami-ongaku-eizo-commercial-grade-cljs-stack.md).

Note/CC/pitch-bend/aftertouch events live on an integer-tick timeline (fixed
PPQ, no floating-point beat math — same discipline real DAW/MIDI internals
use to avoid drift), grouped into loopable patterns/clips and placed on
tracks. Includes:

- `kami.ongaku.sequencer` — event/pattern/track IR, validation, `quantize-events`
  (partial-strength + swing), `apply-groove` (repeating per-slot
  timing/velocity offset templates, i.e. "groove quantize"), and
  `flatten-track` (clip-placed track → absolute-tick event stream).
- `kami.ongaku.sequencer.smf` — Standard MIDI File (SMF, `.mid`) Format 1
  encode/decode: VLQ delta-times, running status, note-on/off pairing,
  tempo/time-signature meta. Operates on plain int vectors (0-255), not
  platform byte types, so it has **zero** reader-conditional branches.

## Not in v0

- No audio rendering/playback — pure event/timing data model + SMF I/O.
- No plugin/instrument routing or mixing — that's `kami-ongaku-project`'s job.
- SMF export flattens clip structure into per-track absolute-tick streams;
  clip/loop boundaries do not survive a round-trip (matching real-world MIDI
  file semantics — SMF itself has no concept of DAW clips).
- Program-change and sysex events are safely skipped on import, not modeled.
- `.cljc` byte-level helpers in `smf.cljc` are written portably for `:cljs`
  but were only exercised under `:clj` (`clojure -M:test`) in this repo's own
  CI so far — no `:cljs` test runner has been wired up yet.

## Usage

```clojure
(require '[kami.ongaku.sequencer :as sq]
         '[kami.ongaku.sequencer.smf :as smf])

(def track
  {:name "piano" :channel 0
   :clips [{:start-tick 0
            :clip {:name "verse" :length-ticks 960 :loop? true
                   :events [{:type :note :pitch 60 :velocity 100 :tick 0 :duration-ticks 240 :channel 0}
                            {:type :note :pitch 64 :velocity 90 :tick 240 :duration-ticks 240 :channel 0}]}}]})

(def flat (sq/flatten-track track))
(def bytes (smf/export-smf {:ppq 480 :bpm 120 :time-signature [4 4] :tracks [flat]}))
(smf/import-smf bytes)
```

## Test

```bash
clojure -M:test
```
