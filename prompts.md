**Bootstrap**: Provide the full file contents for a new static web project with live reload. Create new files if necessary.

* package.json: Must be configured with browser-sync (the latest version) as a dev dependency and an npm run dev script to start the server. This script must watch all .html, .css, and .js files for changes.  
* index.html: A basic starter page, linked to style.css and including main.js (with the defer attribute).  
* style.css: Some simple, obvious styles.  
* main.js: A simple console.log statement to confirm it has loaded.  
* README.md: Simple instructions on how to npm install and npm run dev.  
* .gitignore: exclude `node_modules` and other commonly ignored files.



**Hello Sine\!**: Generate a web app with index.html and main.js to create a sine tone generator.

* index.html:  
  * The big text of "Hello, Sine\!" in the middle of the page.  
  * Below the big text, add a "Start"/"Stop" toggle button.  
  * Include main.js with defer.  
* main.js:  
  * Use Web Audio API: AudioContext, OscillatorNode (440 Hz), GainNode (default gain 0).  
  * Initialize AudioContext on first click.  
  * Bind the button to start() and stop() functions that also toggle the button's label.  
  * start(): Ramp gain to 0.25 over 20ms.  
  * stop(): Ramp gain to 0.0 over 20ms.



### **Random Synth Note App**

**Goal:** Create a web app that plays a single, random note from a subtractive-style synth when a button is held down. The note's "key-down" (ADS) envelope is triggered on press, and its "key-up" (R) envelope is triggered on release.

**File Structure:**

* Create a directory named `random-arp`.  
* Inside this directory, create `index.html` and `main.js`.

#### **`index.html` Specifications**

* Include a main heading (e.g., `<h1>`) with the text "Random Arpeggiator".  
* Include a `<button>` element with the ID `note-button` and the text "Play Note".  
* Link to `main.js` using a `<script>` tag with the `defer` attribute.

#### **`main.js` Specifications**

* **Global State:**  
  * `let audioContext;` (to be initialized on first user interaction).  
  * `let currentVoice = null;` (to track the single active `Voice` instance).  
* **Scale Definition:**  
  * Create an array of 24 integer MIDI pitch values for two octaves of the **C Lydian** scale, starting at **C3 (MIDI 48\)**.  
  * *(Note: The C Lydian scale is C, D, E, F\#, G, A, B. The MIDI values will be: `[48, 50, 52, 53, 55, 57, 59, 60, 62, 64, 65, 67, 69, 71, 72, 74, 76, 77, 79, 81, 83, 84, 86, 88]`).*  
* **`Voice` Class:**  
  * **`constructor(audioContext, midiPitch)`:**  
    * **Goal:** Build the audio graph for a subtractive synth. (e.g. Minimoog and Prophet 5\) This method *builds* the components but does *not* trigger the envelopes.  
    * **Graph Components:**  
      * **Oscillators:** Create two `OscillatorNodes` (`osc1`, `osc2`).  
        * `osc1`: `type: 'sawtooth'`, frequency from `midiPitch`.  
        * `osc2`: `type: 'square'`, frequency from `midiPitch` but slightly detuned (e.g., 5 cents sharp).  
      * **Mixer:** Create `GainNode`s for `osc1` and `osc2` (e.g., set both to `0.5`) and route them to a single filter.  
      * **Filter:** Create a `BiquadFilterNode` (`type: 'lowpass'`).  
        * **Hard-code** its base values (e.g., `frequency: 200`, `Q: 10`).  
      * **Amplitude Envelope:** Create the main `GainNode` (`ampEnvGain`) that controls the voice's final volume. Set its initial gain to `0.0`.  
    * **Audio Graph Connections:**  
      * `osc1` \-\> `osc1Gain` \\  
      * `osc2` \-\> `osc2Gain` / \-\> `filter` \-\> `ampEnvGain` \-\> `audioContext.destination`  
    * **Start Oscillators:** Call `oscillator.start()` on *both* oscillators. They will be silent until `noteOn` is called.  
    * **Store Nodes:** Store `ampEnvGain`, `filter`, `osc1`, and `osc2` as class properties.  
  * **`noteOn(triggerTime)`:**  
    * **Goal:** Trigger the **Attack, Decay, and Sustain** phases. `triggerTime` will be `audioContext.currentTime`.  
    * **Hard-Coded ADS Values:**  
      * `AMP_ATTACK = 0.02`  
      * `AMP_DECAY = 0.1`  
      * `AMP_SUSTAIN = 0.8` (This is the gain *level*)  
      * `FILTER_ATTACK = 0.01`  
      * `FILTER_PEAK_FREQ = 4000`  
      * `FILTER_DECAY = 0.05`  
      * `FILTER_SUSTAIN_FREQ = 1000` (This is the frequency *level*)  
    * **Amp Envelope (ADS):**  
      * Cancel any previous ramps on `ampEnvGain.gain`.  
      * `setValueAtTime(0.0, triggerTime)`.  
      * `linearRampToValueAtTime(1.0, triggerTime + AMP_ATTACK)` (Attack).  
      * `linearRampToValueAtTime(AMP_SUSTAIN, triggerTime + AMP_ATTACK + AMP_DECAY)` (Decay to Sustain).  
  * **`noteOff(releaseTime)`:**  
    * **Goal:** Trigger the **Release** phase. `releaseTime` is the duration (e.g., `0.5`).  
    * `const now = this.audioContext.currentTime;`  
    * **Amp Envelope (Release):**  
      * `this.ampEnvGain.gain.cancelScheduledValues(now)`.  
      * `this.ampEnvGain.gain.setValueAtTime(this.ampEnvGain.gain.value, now)` (Start ramp from current gain).  
      * `this.ampEnvGain.gain.linearRampToValueAtTime(0.0, now + releaseTime)`.  
    * **Filter Envelope (Release):**  
      * `this.filter.frequency.cancelScheduledValues(now)`.  
      * `this.filter.frequency.setValueAtTime(this.filter.frequency.value, now)`.  
      * `this.filter.frequency.linearRampToValueAtTime(200, now + releaseTime)` (Ramp back to base).  
    * **Stop Oscillators:**  
      * Schedule *both* oscillators to `stop(now + releaseTime)` to clean up resources *after* the note has faded out.  
* **Event Listeners:**  
  * Get the button (`#note-button`).  
  * Add a **`mousedown`** listener to the button:  
    * If `audioContext` is not yet created, initialize it (`new AudioContext()`).  
    * Check if `audioContext.state` is `suspended` and call `audioContext.resume()` if needed.  
    * Select a random MIDI pitch from the scale array.  
    * Create a new `Voice` instance with this pitch and store it in `currentVoice`.  
    * Call `currentVoice.noteOn(audioContext.currentTime)` to trigger the sound.  
  * Add a **`mouseup`** listener to the button:  
    * Check if `currentVoice` exists.  
    * If it does, call `currentVoice.noteOff(0.5)`.  
    * Set `currentVoice` back to `null`.  
  * Add a **`mouseleave`** listener to the button:  
    * This event's logic must be **identical** to the `mouseup` listener. This is essential to prevent a "stuck" note if the user holds the button, drags the mouse off it, and then releases.

