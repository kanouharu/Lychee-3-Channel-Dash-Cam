# <p align='center'> [Lychee 3 Channel Dash Cam](https://www.amazon.com/Channel-G-Sensor-Detection-Parking-Monitor/dp/B0BRV1DSS6) Reverse Engineering Journal</p>
**Integrated Circuit** GPCV6248A-QCM51 *(No datasheet!)*
**Flash Chip**: P25Q16SH

## *<p align='center'> 12/ 27 / Reiwa 5 </p>*
* Opened up
+ Photographs Taken 
+ Detatched PCB
+ PCB Underside Photographs
* Initial Research:
    * Unable to find data sheets or user guides for SOC, but the SPI chip has available data sheet
    * TX pad and GND pad located, RX pad not identified. No apparent JTAG connections.
    * Measured some consistent frequency on TX pad using crude oscilloscope. No success reading output [^1]
## *<p align='center'> 12/ 28 / Reiwa 5 </p>*
* Moving on to desoldering SPI flash
* Crude breakout circuit assembled on protoboard,
[^1]: Discovered USB-serial adapter likely inoperable. Switching to FT232RL on [MCU_LAB board!](https://hackerboxes.com/products/hackerbox-0051-mcu-lab) First time using.
## *<p align='center'> 1/ 2 / Reiwa 6 </p>*
* Extracted flash contents (first two attempts failed. Noticed slightly loose jumper wire.)

## *<p align='center'> 1/ 3 / Reiwa 6 </p>*
* Initial application of Binwalk. Flash appears to just store images!

## *<p align='center'> 1/ 8 / Reiwa 6 </p>*
* ~~Just realized I’m not logging anything!~~

* Ghidra happened at some point in the last 5 days. Needed to be compiled to run on Apple Silicon without gatekeeper/code signature checks triggering on libraries and binaries. The alternative would be a very protracted dance with self-signing code manually and that’s just a bad idea. 

* binwalk seems to struggle with parsing and identification, ghidra brought the full contents of the image into grasp. The tool is both intuitive and puzzling, but analyzing [a more familiar firmware image](https://github.com/kanouharu/EiE) helped solidify some basic things. 

* Obviously, there’s code in the image. Curiously, the decompilation starts off with a reset function you’d expect to be there and I notice some arithmetic that maps to exception vectors that are apparently fixed for the ARM7TDMI platform (yay, documentation!) Actually reversing embedded code is not the goal here and well beyond my current skill level.Super fun though. 

* Binwalk at least pointed the way to the section of flash that appears to hold data, rather than code. Ghidra nicely previews the JPGS and a casual scroll shows the sequentially flashed data with roughly 1500 bytes padding them, so I’ll have to be careful overwriting them.

* Ghidra doesn’t appear to intuitively jump to any detected data, but more scrolling revealed the WAV file that plays. Unlike the images, I cannot directly extract WAV files using the GUI it seems.

* *UNIIIIIX.* Not sure if there is a better tool for the job, but the revelation that the `dd` utility allows more granular data manipulation has stuck with me for a long time without any practical use until now. Initial testing just now shows that I can, as hoped, just yank out specific bits in the firmware image as well as the reverse! The best part is that in this context, I can use some esoteric shell substitutions I forgot about to simplify expressing exact hex values for `dd` arguments! *UNIIIIIIX POWER!!!* Super happy.

* Did I mention I’m on cloud 9 right now? I wish I could do this full time 

+ **TODO**  Do I need to research JPEG and WAV file formats? It occurs to me that I don’t know the mechanism the MCU uses to actually handle media files. Whatever libraries were employed would be minimal, stripped down implementations because this is a constrained environment. With the plan being to actually just customize splash screens, backgrounds, and sound effects  on the dash cam, not flashing in a compatible file format would be…bad.  What are the opportunity costs here

* More casual scrolling reveals some interesting strings:
    * References to firmware upgrading?! 
    * “kangkangyoung” is interesting. 誰ですかっ★
	* aRE tHEse deBuG StRiNgS?

* Question: Why is a 1 second long, 16-bit mono and lossless audio file with a 16000hz sampling rate only 25% smaller than an audio file of the same format but 16 times its duration… I would like to know.
    * I know I lack foundational knowledge about the vagaries of file abstractions, but I really have to wonder. 

	* Ocam’s Razor says: Ghidra or I made a mistake somewhere.
		    
    * If I follow the WAV file header information -as presented by a hex editor- and extract the more reasonable 27kb of audio, the wav file is functionally the same as what I extracted using figures from ghidra about the size of the audio data. Am I fundamentally missing something? I suppose reversing tools are as fallible as their makers and users and the sheer complexity of ghidra probably amplify faults! 
		
	* But then I have to consider than anything parsing the audio would just programmatically follow the header information…which isn’t fun. 

* Interesting strings turn up in the extraneous data.GPZP? that appears in other places of the image, suggesting it’s part of some graphics/UI library, but little information is online? Maybe homegrown, Chinese libraries? Cool!

* With the intended audio payload, I mere edited a desired clip down to the same wav format and crudely (with Disk Destroyer, of course) padded the end of track with what I assume is silence to match the exact size of the original audio in the firmware image.

* The JPEG format is less straightforward with embedded thumbnails and geometries I don’t care for. However, parsing for the end of image segment marker `0xFFD9` helped me spot what was clearly an end of the data. Half a megabyte for an SD image on an embedded system was wildly outside the expected range for size. Looking at the raw data, it looks like there are several JPEG headers in the data extracted as a single image, plus what might be values for the system. I wonder if I should slice out more images than what the tools are detecting?

    * Having only used the dash cam for its intended purpose for mere seconds, I can’t say I know what can potentially be changed. This has gone on long enough though :) This could be an endless sinkhole of fun and I haven’t even learned any Armv4T instructions yet!

* Maybe deleting extraneous metadata can shave off a few hundred bytes if GUI image editing tool can get me in the ballpark. No way the MCU is wasting cycles parsing that….But then I’d be throwing off expected lengths of data…hm….I’m tired.

* Working hard for at least 12 hours. Should probably document concretes steps for images…
	1. Open original raw image extraction in hex editor. 
	2. Find `0xFFD9` segment marker of the first jpeg image, thereby determining it’s size in the firmware - albeit crudely.
	3. Use dd to carve out jpg from raw data. Arithmetic expansion! `$(())`
	4. Create suitable imagery that checks all the boxes: Kemonomimi & hacker aesthetics.
	5. Scale down to 320 x 240. Save a version that is less than original jpeg.
	6. Use comment section of jpeg to pad size to match original. No shame. Just do it.
	7. *Pray the firmware isn’t extra picky about parsing jpegs.*

## *<p align='center'> 1/ 9 / Reiwa 6 </p>*

 * 4 custom images and one custom audio file have been prepared, matching the exact size in bytes of the media I’d like to replace in the dashcam firmware. 

 * While I kept learning about audio and image encoding to a minimum because this is really just for amusement and reinforcing skills, I was made superficially aware of different algorithms used in preparing jpegs. The different options were primarily used due to their impact on file size as I required byte-level precision for file sizes. Thank you GIMP image editor! 

* I wonder if this in any way prepared me for playing Binary Golf, which I’d seen a lot of hype about in the nerdiest of nerd circles. Trying different manipulations to hit an exact size feels analogous to playing golf…Though, arguably the only polyglot-esque files would be the erroneous binwalk and ghidra extractions. In any case, it was fun.

* All that’s left is:
	- modifying the firmware image, flashing it
	- soldering the flash chip, the speaker, the battery
	- resembling the device
	-testing it
* If I could afford to invest more time:
    - I’d like to try powering the device outside an automotive context
    - **Finding the holy grail: a JTAG interface**
    - Revisiting the UART issue [^3]
    - Most importantly: software attacks on the thing!

    [^3]: Research suggests there may be a line driver boosting voltage of UART signal to a level that exceeds specified tolerances of the serial-usb adapter. A real logic analyzer or oscilloscope would help! Concept of UART pre-amp is cute.

## *<p align='center'> 1/ 10 / Reiwa 6 </p>*

* The device has been fully assembled. Sadly, first time actual use of the thing suggests that all of my assumptions about when data I’ve manipulated appear to be wrong, *with the notable exception of audio playback*.

* None of the images I’ve added appear on the screen anywhere that I’ve tested. The beach images were presumable for standby screensavers. Standby mode is only vaguely mentioned in the device manual, but would likely be when video recording is pause and there is no playback of video in playback mode. The screen simply turns off after the configured time.
	
* The Goodbye image should appear on shutdown. That was replaced and a black image is shown rather than the custom image.

* There are a few possibilities for that which immediately come to mind:

    * Somewhere in the firmware, there is a pointer to the data section of each respective JPEG. The large amount of metadata is in the firmware image due to laziness and isn’t being parsed at all? There is only enough code to parse that specific section when displaying an appropriate, predetermined (by the manufacturer) image. 
    * My basic understanding of the role the firmware image plays for the device is that, after the firmware linking process, the resulting image structure is written to flash. That structure is not how the image will be loaded into memory. I considered as a precaution trying to find the locations of the JPEGS in other parts of the image, but realized there would instead be calculated offsets, which I wouldn’t find without first reversing other parts of the image to establish the sections explicitly defined in a linker script…or doing more involved physical reversing involving JTAG and stepping through code as the device ran.
	* Could I instead emulate an ARM7TDMI device with dedicated video encoding and display peripherals? That seems like a big ask.
	* If I wanted to find a hidden JTAG connection, presupposing none of the identified pads are used for a minimal 2-wire debug solution, my first suspect might be the rear camera jack? Would the debugging periph share pins with another periph? I feel like of heard of such multiplexing. I suppose documentation would shed some like on what is and is not plausible…but there is no public information on this particular MCU. Hmmm [^2]

	* Whatever is parsing JPEGs is limited such that my comment section padding is a problem. For example, some of the comments involve more than plain ASCII encoded characters. The system supports the Japanese and English languages -which I used- but wouldn’t be as flexible as a less constrained system like a tablet or desktop.  The comment section being present at all could be an issue.
	        
    * The likely interrupt-driven process to display things such the supposed start up and shutdown screens is sensitive to the timing of all the instructions involved. Some assumptions by the manufacturer  might be involved in how long it takes to render an image on screen. An image of all black or white would probably be faster to display vs what I’ve added, despite the size of the total image matching.

* Looking at the comment section of the JPEGS the device itself produces in photo mode…There appears to be something called GPencoder. Quick research led me to a blog by a Japanese nerd who speculates that photomode produces JPGS from AVI recordings using the open source encoder… There are very few references to such a thing online. Mostly people casually mentioning the EXIF data in blogs and forum posts. However, Google’s code archive and open hub both have a project of that name…that was deleted in both cases. 
    * Not to be dissuaded, I checked webarchive.org. There’s only one snapshot that seems to have code for the project and it’s decidedly a java project. The likelihood that java is running in this embedded context is…unlikely…but entirely possible.  I have the code downloaded for further investigation, but that’s just a precaution.
* [^2]:Some quick research confirms that some MCU’s will allow JTAG functionality over pins that have other uses. ESP32 based devices, for example, can be debugged through an SD card slot. Hey, my device has an SD card slot and non obvious JTAG connections! This is bad. 

* Thinking again about Jtag again…and there is noticeably an unused, 6-pin FFC connection on the PCB that I’ve been subconsciously filtering out because I don’t want to think about camera and display technologies at all…But considering the size of the device and possible design constraints…Maybe they would do that? Research suggests that this too is plausible.

## *<p align='center'> 1/ 14 / Reiwa 6 </p>*
+ Beginning to investigate the lone FFC as potential JTAG port!

* Previous poking around with cumbersome multimeter probe tips showed continuity between an outside lead of the FFC connector and a pad on the PCB marked (-), ground. Looks like I found ground, which is pin 6 on a little FFC breakout I purchased!

* Lacking a fancy, expensive tool like a jatagulator, I naturally sought out an alternative. The [BlueTag](https://github.com/Aodrulez/blueTag) GitHub project caught my eye, being a raspberry pico based tag detector. My attempts to detect jtag signals from the unused FFC connection failed. I tried various things, up to and including holding the ‘reset’ switch for at least 30 seconds (debug mode, go!) and tapping the devices ‘ok’ button on the version number menu item as if it were an android phone with developer mode disabled :joy:. If that is a JTAG port, I suspect it’s disabled. 

* To see if that port has any active functionality at all, I suppose I could try an open source logic analyzer based on the pico…At the very least, I expect some signal on the FFC connector. It’s entirely possible this same exact board I’m targeting is used in multiple products, with some actually using the FFC connection. I’d have to actually reverse the firmware image I pulled to figure out where j-tag functionality is hidden. The [hardware hacking handbook](https://nostarch.com/hardwarehacking) has some ideas on that, like debug fuses I can zap with their product placement. 

* Actually, I haven’t thought much about the 2.5mm audio jack used for the dash cam’s rear camera. It was previously mentioned as being a potential debug port. I can’t readily imagine that being the case, but 2-wire debug is actually a thing as are proprietary solutions(Audio plugs typically have 3 signals…I thing 2WD would need at least ground as a reference voltage plus I/O) That would represent a deviation from what looks like the built in default debugging capability of an ARM7TDMI MCU. Would a budget dash cam maker bother? Maybe they’re awash with fancy arm cores.


