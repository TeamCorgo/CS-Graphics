<p align="center">
  <img src="./overhead/web_header.gif" alt="MissionForce: CyberStorm Logo">
</p>

# Graphics

## Cursors
Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩 100% 7/7\
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
Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩 100% 3/3\
Three icon variants have been preserved:
<p align="center">
  <img src="./icons/gog.png" alt="GOG icon" width="64">
  <img src="./icons/goggame-2099484877.png" alt="GOG launcher icon" width="64">
  <img src="./icons/Icon1.png" alt="Original game icon" width="64">
</p>

## Fonts
Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩 100% 3/3\
Fonts are stored in `.FNX` files with a shading, gradient, or shadow effect (fat). Without the gradient applied, the text can be difficult to read. A readable (slim) version is also provided, as modern font formats do not support the gradient effect.

## Images
.ART Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩 100%   1/1\
.ANX Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩⬜ 96%  718/746\
.BMX Progress: 🟩🟩🟩🟩🟩⬜⬜⬜⬜⬜ 58%   89/152
<p align="center">
  <img src="./overhead/Bioderm danger idle.gif" alt="Bioderm face gif" width="128">
</p>


Progress so far: `.anx` & `.bmx` files contain image information with palette information being stored within `.plx` formats. It apears `.anx` format is for my complex visual information such as herc rotations with `.bmx` being used for more static content. The files are stored compressed complicateing the reading process however the issues have been mostly ironed out. 

Next step: Each image file needs to be matched with the correct palette information, although the two are only loosely tied together. It appears that the game selects the appropriate palette based on the current game screen, with the expectation that any newly loaded image content will inherit that palette. Another complication is that effects such as muzzle flashes and the HERCs' blinking lights appear to dynamically alter their palette colors. The image containers can also hold multiple frames of content.

### HERC Lights (complication)
<p align="center">
  <img src="./overhead/HERC lights.gif" alt="HERC Lights" width="256">
</p>
Units use placeholder index colors to represent blinking lights in the game. Because units can have multiple colors and varying pulse rates, this adds complexity to the preservation process.

For the first pass, units will be preserved as they appear in the game files rather than their rendered appearance in-game. A second preservation pass can then be used to store the image data in an “all lights off” state, providing a consistent representation independent of the units’ lighting behavior.

## Color Palettes
.PLX Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩 0% 31/31\
Thirty-one .plx files were exported to the modern `.gpl` **GIMP Palette** format, which is also the palette format used by **Aseprite**.

CS1 `.PLX` are primarily named in "S2P3.plx" format. This represents the second star system `S2` and the third planet `p3`. There is expected to be a large amount of pixel color overlap between files since the palette also controls Unitech & Cybrid unit colors. (This could have been used to tint the color of units depending on the planets atmosphere).

## Tools
### FNX Converter
<img src="./overhead/FNX Converter.png" alt="FNX Converter" width="264">

Used to render and convert `.fnx` fonts into useable formats (`.png` & `.ttf`). 

### ART Converter
<img src="./overhead/ART Converter.png" alt="ART Converter" width="264">

Used to render and convert `.art` fonts into a modern `.png` format. There is a single `.art` assest in the game being `SEQUEL.ART` used to promote Cyberstorm 2 when the player closes the game. This format is unique as it includes palete information where the common `.anx` & `.bmx` require external `.plx` palete files.

### PLX Converter
<img src="./overhead/PLX Converter.png" alt="PLX Converter" width="264">

Used to render and convert `.plx` palettes into the modern `.gpl` **GIMP Palette** format. 

### GIF Animator
Used to convert asset frames into a visual demo. The tool allows users to set the frame rate and specify whether the animation is “stacked.” A stacked animation tracks only the pixel differences from the previous frame, reducing redundant data between frames.

### Resource Hacker (External)
The game cursors were extracted using **Resource Hacker** and converted from `.cur` format to `.png` and `.ico` for easier viewing and use. The original game icon was a hexagon and was embedded within the `.exe` file. **Resource Hacker** was used to extract the icon and save it as a standalone `.ico` file.
