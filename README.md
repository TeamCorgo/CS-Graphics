<p align="center">
  <img src="./overhead/web_header.gif" alt="MissionForce: CyberStorm Logo">
</p>

# Graphics

## Cursors
Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩 100%  7/7\
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
Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩 100%  3/3\
Three icon variants have been preserved:
<p align="center">
  <img src="./icons/gog.png" alt="GOG icon" width="64">
  <img src="./icons/goggame-2099484877.png" alt="GOG launcher icon" width="64">
  <img src="./icons/Icon1.png" alt="Original game icon" width="64">
</p>

## Fonts
Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩 100%  3/3\
Fonts are stored in `.FNX` files with a shading, gradient, or shadow effect (fat). Without the gradient applied, the text can be difficult to read. A readable (slim) version is also provided, as modern font formats do not support the gradient effect.

## Images
.ART Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩 100%    1/1\
.ANX Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩 100%  746/746\
.BMX Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩 100%  152/152
<p align="center">
  <img src="./overhead/Bioderm danger idle.gif" alt="Bioderm face gif" width="128">
</p>

Progress so far: `.anx` & `.bmx` files contain image information with palette information being stored within `.plx` formats. It apears `.anx` format is for my complex visual information such as herc rotations with `.bmx` being used for more static content. The files are stored compressed complicateing the reading process however the issues have been mostly ironed out. 

### HERC Lights (complication)
<p align="center">
  <img src="./overhead/HERC lights.gif" alt="HERC Lights" width="256">
</p>
Units use placeholder index colors to represent blinking lights in the game. Because units can have multiple colors and varying pulse rates, this adds complexity to the preservation process.

For the first pass, units will be preserved as they appear in the game files rather than their rendered appearance in-game. A second preservation pass can then be used to store the image data in an “all lights off” state, providing a consistent representation independent of the units’ lighting behavior.

## Videos
.FLX Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩  100%  256/256\
The game uses `.FLX` files to represent video content, commonly for death animations and screen transitions. However, there does not appear to be a clear distinction between when an animated `.BMX` file is used versus an `.FLX` file. Most files contain and apply their own color palette, while others appear to inherit a palette that has already been applied. Sound and music are not embedded directly within the `.FLX` files. Instead, the game tracks the current playback frame and uses it to trigger the appropriate audio cues. The game uses approximately 15 FPS as the playback speed for its videos.

## Vertex
.PLY Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩  100%  5/5\
`.ply` files contain vertex information used to generate 2D polygons. These polygons were then used to determine which object the player’s cursor was hovering over.

## Color Palettes
.PLX Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩 100%  31/31\
Thirty-one files were exported to the modern `.gpl` **GIMP Palette** format, which is also the palette format used by **Aseprite**.

CS1 `.PLX` are primarily named in "S2P3.plx" format. This represents the second star system `S2` and the third planet `p3`. There is expected to be a large amount of pixel color overlap between files since the palette also controls Unitech & Cybrid unit colors. (This could have been used to tint the color of units depending on the planets atmosphere).

`.BMX` can also embed a `.PLX` file within itself.

## UI positions
.BOX Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩 100%  35/35\
.PLY Progress: 🟩🟩🟩🟩🟩🟩🟩🟩🟩🟩 100%  5/5\
Forty files were exported into `.png` as a visualization.

`.BOX` files contain positional data markers used to place on-screen elements. This is particularly useful for language localization, allowing the positioning of text and other elements to be adjusted for different languages.\
`.PLY` files contain vector information used to define more complex shapes than the rectangular `.BOX` files. The game uses these generated 2D polygons to trigger mouse hover events.

# Tools

## ANX & BMX Converter
<img src="./overhead/ANX & BMX Converter.png" alt="ANX & BMX Converter" width="264">

Used to render and convert `.anx` & `.bmx` all image frames into the modern `.png` format. Lets the user apply a `.plx` color palette of their choice.

## FLX Converter
<img src="./overhead/FLX Converter.png" alt="FLX Converter" width="264">

Used to render and convert `.flx` image frames into the modern `.png` format. Lets the user apply or export a `.plx` color palette.

## FNX Converter
<img src="./overhead/FNX Converter.png" alt="FNX Converter" width="264">

Used to render and convert `.fnx` fonts into useable formats (`.png` & `.ttf`). 

## ART Converter
<img src="./overhead/ART Converter.png" alt="ART Converter" width="264">

Used to render and convert `.art` fonts into a modern `.png` format. There is a single `.art` assest in the game being `SEQUEL.ART` used to promote Cyberstorm 2 when the player closes the game. This format is unique as it includes palete information where the common `.anx` & `.bmx` require external `.plx` palete files.

## BOX Converter
<img src="./overhead/BOX Converter.png" alt="BOX Converter" width="264">

Used to render `.box` (retangular) UI placements into `.png`. 

## PLY Converter
<img src="./overhead/PLY Converter.png" alt="PLY Converter" width="264">

Used to render `.ply` (2D polygon) UI zones into `.png`.

## PLX Converter
<img src="./overhead/PLX Converter.png" alt="PLX Converter" width="264">

Used to render and convert `.plx` palettes into the modern `.gpl` **GIMP Palette** format. 

## GIF Animator
<img src="./overhead/GIF Converter.png" alt="GIF Converter" width="264">

Lets the user export sequenced `.png` image frames into an animated `.gif` video.

## Resource Hacker (External)
The game cursors were extracted using **Resource Hacker** and converted from `.cur` format to `.png` and `.ico` for easier viewing and use. The original game icon was a hexagon and was embedded within the `.exe` file. **Resource Hacker** was used to extract the icon and save it as a standalone `.ico` file.
