# Taurimoji

A simple emoji picker desktop application.

Uses [the complete v17.0 emoji list](https://unicode.org/emoji/charts/full-emoji-list.html) (with a few taken out because they seemed buggy on my machine).

Made by [C. Liam Brown](https://cliambrown.com/)

## Installation

1. Clone the repo to your machine
2. Follow the instructions at https://v2.tauri.app/distribute/#distributing to build an installer
  - `/src-tauri/tauri.conf.json` is currently configured to build only a .deb installer file (for Linux operating systems); you may need to make adjustments for your specific device
3. Install Taurimoji
4. Optional but pretty sweet: Create a custom keybinding to open Taurimoji

## Usage

- Type an emoji name to filter the list of emojis
- Click an emoji to copy it to your clipboard and close the app
- Shift + click an emoji to copy multiple emojis without closing the app
- Tab = highlight the next emoji
- Shift + Tab = highlight the previous emoji
- Enter = copy the highlighted emoji and close the app
- Shift + Enter = copy multiple emojis without closing the app
- Ctrl + W = close the app
- Esc = Clear the filter input

## Built With

- [Tauri](https://tauri.app/)
- [Vue](https://vuejs.org/)
- [Tailwind](https://tailwindcss.com/)

## Disclaimer

This software is provided as-is. Its developer makes no other warranties, express or implied, and hereby disclaims all implied warranties, including any warranty of merchantability and warranty of fitness for a particular purpose.