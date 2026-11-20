# Hue · Scoring and game logic

Hue combines established color/audio mathematics with app-specific scoring curves and round rules.

## 1. Perceptual color scoring

The color utility converts sRGB to linear RGB, then XYZ and CIE LAB. It calculates **CIEDE2000** difference between the target and the guess. This is the color distance used by the app, rather than a direct distance between RGB channels.

The distance is mapped into a bounded game score:

`accuracy = clamp(100 - 3.4 × distance^0.82, 0, 100)`

The game can scale the distance by its forgiveness setting before applying that curve. CIEDE2000 is an established formula; the curve and its placement in the memory/recreation loop are product choices.

## 2. Pitch matching in cents

Pitch error is calculated as `abs(1200 × log2(guessFrequency / targetFrequency))`. Thus the same frequency ratio produces the same error at different absolute frequencies.

The app divides the error by its forgiveness factor and maps it with `100 / (1 + adjustedError / 420)`, clamped to 0–100. A perfect ratio gives 100; increasing error reduces the score gradually.

## 3. Procedural tones with smoothed controls

`AVAudioSourceNode` fills the audio buffer with sine samples. A phase accumulator advances by `2π × frequency / sampleRate` for each sample.

Instead of jumping immediately to a newly selected frequency or amplitude, the generator moves the current value a small fraction toward its target each sample. In the implementation those fractions are 0.006 for frequency and 0.018 for amplitude. This gives dragged pitch controls and starts/stops a gradual transition. The tones are generated at runtime rather than selected from a recording for every possible pitch.

## 4. Shape scoring with symmetry

Shape accuracy combines position, relative size, rotation and—when relevant—rectangle aspect ratio. Weights are normalized after omitting dimensions that do not apply.

Rotation error is folded into the target shape’s symmetry period: for example, triangle 120°, rectangle 180°, pentagon 72° and hexagon 60°. Circle and square rotation are ignored by this implementation. The dimensional score uses a 1.35 exponent; a different shape type applies a 0.35 multiplier. Forgiveness adjusts the resulting error.

This avoids treating a symmetry-equivalent orientation as a large rotation error, while still considering placement and size.

## 5. Timing precision

The time mode compares held duration with the target and computes percentage error. The score decreases with that error, scaled by forgiveness. Using relative error makes the result depend on the target duration rather than only the raw number of milliseconds missed.

## 6. Repeatable daily targets

The server hashes the date into a numeric seed and feeds it to a 32-bit linear congruential generator. Its output builds the daily target sequence. The generated rounds are also saved in the daily challenge table.

Same date and generator logic yield the same generated target values. This is deterministic content generation, not a cryptographic random-number design. It does not by itself imply that every client-side shape detail is specified in the daily payload.

## 7. Matchmaking, round completion and ranking

The queue searches for another waiting player in the requested mode. If none exists, it stores the player. The search is linear in queue size; there is no skill-rating matcher in this component.

Rounds collect one answer per player in a map. Completion persists answers with a unique round/user key; a missing response can become a zero-score answer. Continue/readiness sets coordinate the next phase.

For daily scores, the service retains the better submission and calculates a personal rank as `number of strictly higher scores + 1`. The top-list query also orders equal scores by update time, so its sequential row labels and personal tie ranks serve slightly different purposes.

[Architecture](ENGINEERING.md) · [Data model](DATA_MODEL.md)
