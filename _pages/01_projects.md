---
layout: page
title: Projects
permalink: /projects/
order: 1
published: true
---

<h2 style="border-bottom:1pt solid #E8E8E8;">Blog</h2>

<ul>
  {% assign sorted = site.ramblings | sort: 'order' | reverse  %}
  {% for ramble in sorted %}
    <li>
      <a href="{{  ramble.url | relative_url  }}">{{ ramble.title }}</a>
    </li>
  {% endfor %}
</ul>

<h2 style="border-bottom:1pt solid #E8E8E8;">Recent Tinkerings</h2>

### [wasm-audio-worklet](https://github.com/JamieBeverley/wasm-audio-worklet)
- A template/to-be-tutorial/package for writing audio DSP in Rust, compiled to WASM for use with the Web Audio API as an `AudioWorkletNode`
- Quickly getting side-tracked into a granular synthesizer `npm` package because I can't help it
- Maybe one day spun off as a `wasm` polyfill/alternative to `AudioBufferSourceNode` to account for some of its less-than ideal quirks? e.g. limitation around re-using nodes leading to lots of JS object instantiation/GC in the main thread if you're trying to do something like granular synthesis

### [this tiny bug fix for python keyring](https://github.com/jaraco/keyring/pull/699)
- `WinVaultKeyring.get_credential("service", "<username_dne>")` with a non-existent username would in some cases return credentials of another username.


<h2 style="border-bottom:1pt solid #E8E8E8;">Published Open Source</h2>

### [react-auto-scroll-table](https://github.com/JamieBeverley/react-auto-scroll-table/) ([npm](https://www.npmjs.com/package/react-auto-scroll-table))
- A React component for airport-style infinite smooth-scrolling displays
- [demo site](https://jamiebeverley.github.io/react-auto-scroll-table/)

### [Estuary](https://github.com/dktr0/estuary) (contributor)
- Available [here](https://estuary.mcmaster.ca/)
- Web-based environment for programming music and visuals.

### [Inner Ear](https://github.com/dktr0/InnerEar) (contributor)
- Web-based ear training exercises
- Developed in Haskell (Reflex, Relfex-DOM) and JS (Web Audio API)

### [Web Dirt](https://github.com/dktr0/webdirt) (contributor)
- JS (Web Audio API) implementation of the [Dirt](https://github.com/tidalcycles/Dirt)/[SuperDirt](https://github.com/musikinformatik/SuperDirt) sound sampler.

<h2 style="border-bottom:1pt solid #E8E8E8;">Past Projects/Maintenance</h2>

### [MIDIPlex](https://github.com/JamieBeverley/MIDIPlex) 
- React Native mobile application sends MIDI messages over Bluetooth Low Energy to an Adafruit microprocessor mounted in a 3D-printed euro-rack module.
- A work in progress - development paused when I fried my microprocessor while soldering...

### [DeadCode](https://github.com/JamieBeverley/DeadCode) ([short demo](https://www.youtube.com/watch?v=kuJlpd2i25k))
- Web-based music performance interface written in React/Redux (frontend) and Node.js (backend).
- Organization of live-coded musical and visual code snippets into a performance-oriented UI.
- Synchronized state over websockets for collaborative performances and suppporting different form-factors (tablet, desktop, hardware MIDI controller) 
- Unfortunately deprecated! Would live to revive this some day, probably as a fresh rewrite in typescript and implementing server synchronization more intelligently...

<h2 style="border-bottom:1pt solid #E8E8E8;">Other Things</h2>

### [MIDISynthDef](https://github.com/JamieBeverley/MIDISynthDef) 
- Small extension for SuperCollider for authoring audio synthesis patches (SynthDef’s) that are responsive to MIDI messages (without all the boilerplate of writing MIDIDefs)

### [Seventeen](https://github.com/JamieBeverley/seventeen)
- Silly, rough, and experimental web audio drum machine designed for tablet
- Sounds fetch from freesound.org, forward/backward/sped/slowed-down sequencer, infinite seq duration, etc...

### [Pi-Sound Patches](https://github.com/JamieBeverley/pisound-patches)
- SuperCollider effects patches for the [Pisound](https://blokas.io/pisound/)

### [Surfeur Chrome Extension](https://github.com/jamiebeverley/surfeur) 
- Prototype Chrome plugin that augments web browsing with mindful and playful audio-visual experiences. Inspired by slowness and the ['flâneur'](https://en.wikipedia.org/wiki/Fl%C3%A2neur)
- JavaScript (p5.js, Web Audio API, Jquery), HTML, CSS.
