# ⚔️ Sword Duel Alarm

A single-page web app for an always-on Android phone. It watches a camera for someone picking up a foam sword and raising it, then plays a song, sends a push notification, and records the duel until a sword is raised again in victory.

Everything lives in `index.html`. There is no backend. The pose model (MediaPipe Pose Landmarker, lite) loads from a CDN on each page load.

## How detection works

Two modes, chosen in the **Detection** section:

- **Lit sword (default).** The swords light up, so the app looks for pixels close to a sampled glow color and tracks them with a detection box. "Grabbed" means the glow is visible. The trigger gesture is then one of:
  - **Upright, then sideways (default).** Hold the sword straight up and down for the hold time, then within a few seconds turn it flat and hold again. The app works out the sword's angle from the spread of glowing pixels, so a tilted sword reads as "tilted" rather than fooling a plain box check. Tune the angle tolerance, the minimum elongation (a lit sword seen end-on is too round to have an angle) and the time allowed between the poses.
  - **Raise above the shoulder line.** The top of the glow must be above the shoulder line (from the pose model) or, when no shoulders are visible, above the fallback raise line you set with a slider.

  Victory is still a raise: the sword comes down after the opening, then is raised above the shoulder line and held.
- **Pose: grab from rack, then raise.** You draw a box over the sword rack. A wrist entering the box starts a short window. If that same wrist rises above its shoulder within the window and stays there, the alarm triggers. Use this for unlit swords.

After the trigger, the app is in a **duel** phase. A sword that comes down and then is raised and held again (default 2 s) after the minimum duel length (default 30 s) counts as a **victory**. Victory sends a second notification, ends the recording, and handles the music according to the **Duel & victory** settings:

- The song starts from the beginning at the trigger. While the duel is on, each time it reaches the **victory point** (default 4:05) it fades out and fades back in at the **loop start** (default 0:15), so the music keeps going for as long as the duel does.
- At victory the song fades out, jumps to the victory point, fades back in, and plays through to the end. You can instead choose to stop the music or leave it alone.
- The fade length is a slider (default 1 s). **Preview victory jump** and **Preview loop seam** let you hear both transitions without starting a duel. The duel also ends after the maximum duel length or when nobody has been in frame for a minute.

State labels shown on screen: `IDLE → GRABBED → RAISED → TRIGGERED → DUEL → VICTORY → COOLDOWN`.

## Hosting on Vercel

The site is static, so no configuration is needed.

```sh
npm i -g vercel          # once
cd sword-duel-alarm
vercel --prod            # follow the prompts the first time
```

Vercel prints an `https://…vercel.app` address. Open that address in Chrome on the phone. HTTPS is required, otherwise Chrome refuses camera access.

GitHub Pages or Netlify Drop also work: upload `index.html` and open the https address.

## First-time setup on the phone

1. Open the page in Chrome and allow the camera.
2. **Audio:** pick an MP3. It is cached on the phone, so it survives a reload.
3. **Notification:** note the ntfy topic (a random name is filled in). You can change it to something memorable.
4. **Detection:**
   - Lit mode: turn a sword on, hold it in view, tap **Pick sword color**, then tap the glowing part on the video. The debug line shows how many pixels match. Raise the sword and confirm the glow box turns green above the dashed line.
   - Pose mode: tap **Set rack zone** and drag a box over the rack.
5. Tap **Arm**. This unlocks audio playback and the sword sound engine (browsers block sound that was not started by a tap) and requests a screen wake lock. Safari is stricter than Chrome here: everything that needs the tap starts in the same instant, and if the song briefly plays during the unlock the app stops it again within a second.

After a reload, everything is remembered and only the **Arm** tap is needed.

## ntfy app setup

1. Install the ntfy app from Google Play or the App Store.
2. Tap **+** and subscribe to the topic shown on the page (for example `sword-k7m2p9x3qa`). Public ntfy.sh topics are open to anyone who guesses the name, so keep the random name or choose something hard to guess.
3. On the page, tap **Send test notification** and confirm it arrives.
4. In the ntfy app, open the topic's settings and turn on **Instant delivery** so alerts are not delayed by battery saving.

## Keeping the phone awake

The page requests a screen wake lock, but Android can still override it. For an always-on setup:

1. Enable Developer options: Settings → About phone → tap **Build number** seven times.
2. Settings → System → Developer options → turn on **Stay awake** (screen never sleeps while charging).
3. Keep the phone plugged in and leave Chrome in the foreground. Turn off battery optimization for Chrome if the page still gets paused.

## Camera placement

- Put both the sword rack and the area where people will stand in the frame.
- Mount the phone at about chest height, rear camera facing the room, 2 to 4 m from where people stand. Pose detection is best when the whole upper body is visible.
- Avoid pointing the camera at windows or bright lights. In lit mode, dim light actually helps, because the glowing sword stands out more.
- Pose mode only: make sure the rack box does not overlap where people normally stand, or hands will "grab" by accident.

## Testing

The **Test & debug** section has everything for checking detection without annoying the household:

- **Test mode** is a dry run: no music, no notification, no saved recordings, and the cooldown and minimum duel length drop to 5 s. Every event still flashes on screen and appears in the log.
- **Video source** can be the rear camera, the front camera (handy on a laptop) or a video file you recorded earlier.
- **Debug overlay** draws the skeleton, the four landmarks that matter (wrists and shoulders), the glow box and the raise line.
- **Test trigger** and **Test victory** fire the events by hand so you can check the song, the notification and the recording flow.

## Sword sounds

While a lit sword is in view, the app plays a hum, swing sounds that follow how fast each sword moves and turns, and a clash when the blades meet.

Both swords glow the same color, so the app splits the matching pixels into separate connected blobs each frame and tracks up to two of them. Each blob gets its own speed and turn rate. A clash plays on the frame two tracked swords merge into one blob while at least one of them is moving. While the blades stay crossed they read as one sword, which is fine because only the moment of contact matters. An optional second clash trigger fires when a fast-moving sword stops abruptly.

The trigger gesture and victory always use the largest blob.

- **Tracking rate.** The color tracker runs on every camera frame (about 30 fps) while the pose model runs at about 10 fps, so the swing sound lags the motion by only a few frames. The status badge shows both rates.
- **Sounds.** With no files loaded, everything is synthesized in the browser. You can instead load a ProffieOS-style sound font: select the files `hum.wav`, `swing01.wav`…, `clash01.wav`…, optional `swingl01.wav`/`swingh01.wav` pairs for smooth swings, and `in.wav`/`out.wav` for ignition and retraction. Royalty-free sources include the Krotos lightsaber pack, Pixabay, and Freesound filtered to CC0. The font is cached on the phone.
- **Unlocking.** Browsers block sound until a tap. Arm, or any of the sound test buttons, unlocks it. Effects play in test mode too, so you can tune them without arming.
- **Tuning.** The two "full swing" sliders set how fast a move or turn must be for the swing sound to reach full strength. The live line under the video shows how many swords are in view and each one's measured speed and turn rate, so wave a sword and pick values a little under what a real swing reads. "Clash when the two swords meet" can be switched off, and abrupt-stop sensitivity 0 turns that second trigger off.
- **Test swing / Test clash / Test hum** play each effect by hand.

Use the phone speaker or a wired speaker. Bluetooth adds 100 to 200 ms of delay, which makes the swing sound trail the sword.

## Recordings

With **Record duels** on, the app starts recording whenever someone is in frame and keeps the file only if a duel happens. Recording stops at the victory raise. Files are stored in the browser (IndexedDB) and listed under **Recording** with Play (inline preview), Download and Delete buttons. Chrome on Android saves them as MP4 or WebM depending on what the device supports. Keep an eye on the storage line, a 10-minute duel is roughly 90 MB at the default bitrate.

Turn on **Record microphone audio too** if you want sound. It asks for microphone permission and restarts the camera.

## Tuning

| Setting | What it does |
|---|---|
| Color tolerance | How far a pixel may be from the sampled color and still count. Raise it if the glow is missed, lower it if walls or clothes match. |
| Minimum glow pixels | How many matching pixels in the largest connected piece mean "a sword is lit". Watch the live count with the sword on and off and pick a value in between. |
| Join glow pieces closer than | A bright blade often shows as a white core with colored edges, which would split into pieces. Pieces closer than this many pixels (on the 320-pixel-wide analysis frame) count as one sword. If the readout shows one sword as several pieces, raise it; if two separate swords merge too early, lower it. |
| Fallback raise line | Used only when no shoulders are visible. The glow top must be above this line to count as raised. |
| Raise hold time | How long the raise must last before triggering. |
| Grab → raise window | Pose mode only. Time allowed between leaving the rack and raising. |
| Landmark visibility threshold | Minimum confidence for wrists and shoulders. Lower it in dim rooms, raise it if ghost limbs appear. |
| Cooldown | No re-trigger for this long after a trigger. |
| Victory raise hold, minimum and maximum duel length | Control when a duel is considered over. |
