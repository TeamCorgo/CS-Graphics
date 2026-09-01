<p align="center">
  <img src="./images/web_header.gif" alt="MissionForce: CyberStorm Logo">
</p>

# Graphics

## Cursors
Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩 100%
The game cursors were extracted using **Resource Hacker**. They were converted from `.cur` format to `.png` for easier viewing and use.

Seven cursor variants have been preserved:

<p align="center">
  <img src="./cursors/Cursor2.png" alt="Cursor 2" width="32">
  <img src="./cursors/Cursor3.png" alt="Cursor 3" width="32">
  <img src="./cursors/Cursor4.png" alt="Cursor 4" width="32">
  <img src="./cursors/Cursor5.png" alt="Cursor 5" width="32">
  <img src="./cursors/Cursor6.png" alt="Cursor 6" width="32">
  <img src="./cursors/Cursor7.png" alt="Cursor 7" width="32">
  <img src="./cursors/Cursor8.png" alt="Cursor 8" width="32">
</p>

## Icons
Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩 100%
The original hexagonal game icon was extracted using **Resource Hacker**.  
In addition, the GOG release includes its own updated icon set, with the raw `.ico` files provided directly in the installation directory, making extraction unnecessary.

Three icon variants have been preserved:

<p align="center">
  <img src="./icons/gog.png" alt="GOG icon" width="64">
  <img src="./icons/goggame-2099484877.png" alt="GOG launcher icon" width="64">
  <img src="./icons/Icon1.png" alt="Original game icon" width="64">
</p>

## Fonts
Progress: 🟩🟩⬜⬜⬜⬜⬜⬜⬜⬜ 20%

Progress so far: The files `FONT0.FNX`, `FONT1.FNX`, `FONT2.FNX`, `FONT3.FNX`, `FONT4.FNX`, and `FONT10.FNX` are likely font data for the game.

Next step: Decode the .fnx format and preserve it in a modern, accessible format.

## Images
Progress: 🟩🟩⬜⬜⬜⬜⬜⬜⬜⬜ 20%

Progress so far: `.anx` & `.bmx` files contain image information with palette information being stored within `.plx` formats. It apears `.anx` format is for my complex visual information such as herc rotations with `.bmx` being used for more static content. The files are stored compressed complicateing the reading process however the issues have been mostly ironed out. 

Next step: Each image file needs to be matched with the correct palette information, although the two are only loosely tied together. It appears that the game selects the appropriate palette based on the current game screen, with the expectation that any newly loaded image content will inherit that palette. Another complication is that effects such as muzzle flashes and the HERCs' blinking lights appear to dynamically alter their palette colors.

