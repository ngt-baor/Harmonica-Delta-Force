# Harmonica Delta Force

**Version 0.2.2**

A Windows desktop app for managing harmonica scores and playing them in Delta Force.

## What's new in 0.2.2

- The Compact Dock groups speed, playback, and add-music controls into a balanced layout.
- The Play and Stop buttons are equal-sized and centered; disabled button text is easier to read.
- Mouse-wheel scrolling in the song list moves about one song card per notch for smoother browsing.
- The downloadable app is smaller because optional audio-to-score inference and GAME model files are omitted.

## Download and install

[Download the latest Windows Setup installer](https://github.com/ngt-baor/Harmonica-Delta-Force/releases/latest/download/Harmonica.Delta.Force.exe). Run Setup to install the app for your Windows user. It creates a Start Menu shortcut; a desktop shortcut is optional. Python is not required.

Choose a song from the library, or import a MIDI (`.mid` / `.midi`) or JSON score. Use **Nhập bản số** to add numbered notation. Adjust playback speed from the Compact Dock.

- **F8**: pause or resume playback
- **F9** or **Esc**: stop playback and release held keys

The app keeps its settings and music library under `%APPDATA%\GTIHarmonica`. Updating or uninstalling the app keeps this user data. For portable use, create an empty `portable.txt` beside the installed executable.

The lightweight desktop release focuses on score import and playback; audio-to-score conversion and its GAME models are not included.
