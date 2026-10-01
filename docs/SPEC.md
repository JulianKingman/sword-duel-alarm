# Sword Duel Alarm: Build Spec (Pose-Based)

**Goal:** A single-page web app that runs in Chrome on an always-on Android phone. It uses MediaPipe Pose to detect someone grabbing a foam sword and raising it into a fighting stance, then plays a user-supplied MP3 and sends a push notification via ntfy.sh.

**Constraints**
- One self-contained `index.html` with inline JS and CSS. Load MediaPipe Tasks Vision (`PoseLandmarker`, lite model, GPU delegate with CPU fallback) from jsDelivr. No backend.
- Must be served over HTTPS (camera requirement). Deploy target: GitHub Pages or Netlify Drop.
- Mobile-first UI with large tap targets.

**Detection logic**

Pose can't see the sword itself, so a trigger is a "grab then raise" sequence:
1. **Rack zone:** The user drags a rectangle over the sword rack during setup.
2. **Grab:** Either wrist landmark (15/16) enters the rack zone with visibility above 0.6.
3. **Raise:** Within 3 s of a grab, that same wrist rises above its shoulder (11/12) and stays there for at least 500 ms.
4. **Trigger** fires only when grab and raise both happen in sequence. This filters out people walking past or reaching up for unrelated reasons.
5. Run inference at about 10 to 15 fps on a downscaled frame and smooth the landmarks over 3 frames.
6. **Optional confidence boost (toggle):** Sample the sword color at calibration, and require matching pixels near the raised wrist.

**Features**
1. **Camera:** Rear camera (`facingMode: "environment"`) with a live preview and a skeleton overlay.
2. **Audio:** A file picker loads an MP3 (user-provided). An "Arm" button unlocks audio playback, a browser autoplay requirement. The song plays from the start when triggered.
3. **Notification:** POST to `https://ntfy.sh/<topic>` with title "⚔️ Duel detected" and priority high. The topic is a text input, prefilled with a random string.
4. **Cooldown:** No re-trigger for 2 minutes (configurable). Include a "Stop music" button.
5. **Settings:** Sliders for the grab-to-raise window, the raise hold time, the visibility threshold, and the cooldown.
6. **Stay awake:** Request the Wake Lock API, and re-request it on `visibilitychange`.
7. **Persistence:** Save the zone, thresholds, and topic to `localStorage`. Cache the MP3 in IndexedDB so only the Arm tap is needed after a reload.
8. **Debug overlay:** Show the rack zone, a state-machine label (Idle → Grabbed → Raised → Triggered → Cooldown), live wrist and shoulder positions, and fps. Add a "Test trigger" button.

**Acceptance tests**
- Reaching into the rack and raising an arm above the shoulder within 3 s triggers music and a notification within about 1 s of the raise.
- Walking past the rack, or raising an arm without reaching into the rack first, does not trigger.
- It works with the person 1 to 4 m from the camera in normal indoor lighting.
- It sustains at least 8 fps on a mid-range Android phone.
- After a page reload, settings and the MP3 persist and only the Arm tap is required.
- The page runs for at least 1 hour plugged in without the screen sleeping.

**Deliverables:** `index.html` plus a short README covering hosting steps, ntfy app setup, the Android "Stay awake" developer option, and camera placement tips (rack and standing area both in frame, camera at chest height).

---

## Addendum (2026-10-01): lit swords, recording, test mode, Vercel

Changes requested after the original spec:

**Lit-sword color detection is the primary mode.** The foam swords light up, so a lit sword is a bright, saturated color blob that is much easier to find than a wrist entering a rack zone. The app offers two detection modes:
- **Lit sword (default):** Sample the glow color once. Each frame, count matching pixels on the downscaled frame and find the blob's top edge. "Grabbed" = glow visible. "Raised" = blob top above the shoulder line (from pose) or, when no pose is visible (dark room), above a configurable fallback line. Held for the raise-hold time → trigger.
- **Pose grab + raise:** The original rack-zone sequence, kept as a fallback for unlit swords.

**Victory and the duel phase.** After the trigger, the state machine enters a DUEL phase. A sword raised again and held for the victory-hold time (default 2 s), after a minimum duel length (default 30 s), ends the duel ("victory"). The duel also ends after a maximum length (default 10 min) or when nobody is in frame for 60 s.

**Recording.** Record from the camera stream with MediaRecorder:
- Start a pre-roll recording when someone is in frame (pose visible, or a lit sword visible).
- Discard the pre-roll if no duel starts within 3 min or nobody is in frame for 15 s.
- Keep recording through the duel until victory or duel end, then save to IndexedDB. The Recordings list offers download and delete.
- Optional microphone audio (off by default).

**Test mode.** A dry-run toggle: no notification, no music, recordings discarded, cooldown and minimum duel length shortened to 5 s, every event flashed on screen and logged. Source selector: rear camera, front camera, or a video file, so pose and color detection can be checked on a laptop or with a recorded clip.

**Hosting.** Deploy to Vercel (static site, zero config).

## Addendum (2026-10-01): victory point, two-pose trigger, sword sounds

- **Music:** song starts from 0 at trigger; while the duel is on it loops from the victory point (4:05) back to the loop start (0:15) with volume fades; at victory it fades to the victory point and plays through.
- **Trigger gesture (lit mode default):** hold the sword upright, then turn it sideways. Orientation comes from the principal axis of the glowing pixels (angle + elongation), smoothed over 3 frames. The raise gesture remains an option and is still used for victory.
- **Sword sounds:** color tracking runs per camera frame (pose stays at ~10 fps). Speed (frame widths/s) and turn rate (deg/s) drive a Web Audio engine: hum loop, SmoothSwing-style swing layers, swing accents, and an abrupt-stop clash (placeholder until two-color tracking). Synthesized by default; accepts a ProffieOS-style sound font. Test swing / clash / hum buttons.
- **Next:** two-color tracking (one color per sword) for clash-on-contact.
