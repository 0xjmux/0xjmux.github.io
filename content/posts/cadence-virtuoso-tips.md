---
layout: post
tags: ["Blog","Old","VLSI"]
title: "Cadence Virtuoso Tips"
summary: "Usage tips for the Cadence Virtuoso VLSI software for students in EECS119"
date: 2023-05-18T17:59:35-08:00
math: false
draft: false
---


These are my personal notes from taking EECS119 and using Cadence Virtuoso. The way most tutorials show you how to do it is bizarre, and not at all how the software is supposed to be used. I had to figure a lot of this out by trial and error, and wanted to save others the trouble for the future. This obviously doesn't cover everything and I'm still very much a beginner when it comes to using Cadence, but should be helpful for those in EECS119 or just starting out with VLSI. 


---

### General
* Push `v` for vertical cursor in ADE visualization window
* Directory for loading techfile on the EECS servers for EECS119 & EECS170C labs
  * How to find the path of your home directory: `cd ~/ && pwd`
```
[YOUR HOME DIRECTORY]/eecs119/ncsu-cdk-1.6.0.beta/techfile
```
 
### Schematic
* Connect outputs of circut testbench to the `noConn` component from the `basic` library. Eliminates DRC error about unconnected node. 

### Layout
<!-- this was a callout -->
**Most importantly: Do DRC Regularly and _NEVER_ do layout before testing your schematic in simulation**

* Use wires (*Not Rectangles*) to connect components/instances: `ctrl+shift+w`
* Insert your custom components like normal components into the layout, and connect them by drawing onto their metal traces. As such, when you draw your inner components leave space at the end of the trace the pins are on to connect to. 
* To view location of errors in Layout DRC: Verify -> Markers -> Find Markers
* *LAYOUT EZ MODE*
    * From schematic, open LayoutXL
    * In layout XL, go to bottom of screen and click on "generate all from source"
    * LayoutXL will autogen pins, instances of nmos/cmos, and all you have to do is line them up
    * To the right of that button, "Show/Hide incomplete nets" button. Select everything, and then click that to generate lines that go between the things you need to connect. 
    * See screenshots below for more information. 
* One issue with generating from source is it sometimes won't recognize that certain nodes on the circuit are connected, and will keep the lines between them alive even though they're connected. 
* LVS help - "Check against source"
    * Bottom row - green checkmark button. Extracts and checks against schematic, but highlights and tells you what's wrong. Doesn't get all errors, and can be tempermental. 

---

## Screenshots
All of the below Software screenshots were taken in LayoutXL.

#### Create wire
{{< figure width="500" src="https://user-images.githubusercontent.com/20913473/240716733-f27c20d0-d21c-4228-b3dd-599ba8c4714a.png" >}}

#### Generate all from source
 {{< figure width="500"  src="https://user-images.githubusercontent.com/20913473/240716751-d43d4a55-ed85-47cf-ad0b-d0fd39c46a09.png" >}}

#### Place components as in schematic
 {{< figure   width="500" src="https://user-images.githubusercontent.com/20913473/240716783-f030989e-6b14-41cd-a600-fd067ac0fd59.png" >}}

#### Show/hide selected incomplete nets
* This is one of the most useful options in LayoutXL. It creates a ratsnest between your traces similar to how PCB layout works, making connecting your components much easier. 
 {{< figure width="500" src="https://user-images.githubusercontent.com/20913473/240716795-d7348b89-18bf-4884-925c-201713b1eccd.png" >}}

#### Layout showing NAND gates placed as components in a standard cell layout, with wires going between them. 
 {{< figure width="500" src="https://user-images.githubusercontent.com/20913473/240716770-f4663b4b-a879-46e3-9353-dbbc62d740b6.png" >}}

#### 3 input JK Flip-flop (used in EECS119 HW3)
 {{< figure width="500"  src="https://user-images.githubusercontent.com/20913473/240716335-e335b68e-94aa-4ba9-a031-2b659f5387e1.gif" >}}

#### fish
 {{< figure width="500" src="https://user-images.githubusercontent.com/20913473/240716311-bdd6e1b1-c596-4570-a87e-1bf5b7f32264.jpg"  >}}

### Helpful Links
* [Guide to Drawing Clean Schematics with Virtuoso](https://www.egr.msu.edu/classes/ece410/mason/files/guide-schematictips.pdf)
* [Cadence Troubleshooting Guide](https://www.egr.msu.edu/classes/ece410/mason/files/guide-troubleshooting.pdf)
