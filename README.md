# Sound Latency Tester

A simple Nuxt application for measuring microphone and audio output latency in the browser.

The app plays short test sounds, detects the returned audio through the microphone, and calculates latency results from repeated measurements. It can be used to compare headphones, speakers, microphones, or different audio output settings.

## Features

- Microphone and audio output device selection
- Detection threshold control
- Live microphone level meter
- Individual latency measurements and average result
- Short microphone recording, playback, and download
- Reset measurements and refresh device list

## How It Works

The app asks for microphone access, plays a short test sound through the selected output device, and listens for that sound through the selected microphone. The time between playing the sound and detecting it from the microphone is recorded as the latency. Repeating the test gives a list of measurements and an average result.

## Setup

Install dependencies:

```bash
npm install
```

Start the development server:

```bash
npm run dev
```

The app runs at `http://localhost:3000` by default.

## Scripts

```bash
npm run dev
npm run build
npm run preview
npm run lint
npm run typecheck
```

## Notes

- The browser must be allowed to access the microphone.
- Recording playback requires browser support for the MediaRecorder API.
- For best results, use an environment where the microphone can clearly hear the test sound.
- Audio output device selection may not be supported in every browser.
