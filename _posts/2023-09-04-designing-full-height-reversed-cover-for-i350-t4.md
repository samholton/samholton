---
layout: post
title: "Designing Full Height Reversed Cover for i350-T4"
date: 2023-09-04 20:28
categories: ['3D Printing']
tags: [fusion360, design]

image:
  alt: 3d print on printer bed
  path: /assets/images/headers/i350-t4-cover-reversed.webp
  lqip: data:image/webp;base64,UklGRkAAAABXRUJQVlA4IDQAAACwAQCdASoQAAgABUB8JaQAAjzv+cWwAP7v1GWllZBPu2QhOulxzo5ou6W2wlgeAFxQyAAA
---

I recently purchased three Dell i350-T4 to add to my Proxmox nodes. The nodes are in generic 2U cases I got off EBay so I ordered the low profile cards. Turns out, the case I have uses full sized PCIe turned sideways (yes, I built them... no, I didn't check or remember before my late night order).

Easy, I'll just hop on Thingiverse and grab a full size STL. That along with a PCIe riser/ribbon cable, I'll be good to go. After another quick trip to EBay and the print, I was ready to install the card (again). Turns out the orientation of the card with the extender doesn't fit, the NVME PCIe card I'm using for my Longhorn volume is in the way.

At this point, I realize I should have done some more planning. But open up Fusion360 to continue down the rabbit hole. The goal is to design a full sized, reverse mounted cover. In addition, it needs to be rigid enough to support a card installed not into the motherboard, but a ribbon riser cable. Otherwise, the pressure when inserting network cables would push the card into the case.


## Designing the Model
I'm fairly new to Fusion360, coming over from [FreeCAD](https://www.freecad.org/). But overall, the process took around 10 iterations to reach the model I settled on. Some issues along the way:

* The cover not accounting for the fact I was using a ribbon riser and did not have mother board support when pushing in on the card
* The cover not being rigid enough and deforming when plugging in network cables
* The cover being _too_ thick and not being able to access the release tab on the network cable to remove the cable
* The cover being too thick to fit under the retaining piece

I find it very useful to utilize parameters inside Fusion360 - especially when tweaking measurements after each iteration. I got a bit lazy at the end, so I'm not sure all of the measurements correctly adjust or calculate.


## Printing the Model

Generally, my goto plastic is PLA. But I used PETG for this print to avoid any potential heat issues (shouldn't be a concern), and to give the thin areas a bit more strength. PETG layers don't seem to delaminate as easily as PLA. Of course this means more fun when removing the support material. Speaking of support material, this is not a very 3d printer optimized part. I briefly played with print orientation on the first iteration but never revisited it in later iterations to see if it could be improved.

![i350-t4 on print bed](/assets/images/posts/i350-t4-on-print-bed.webp){: width="400" height="400" lqip="data:image/webp;base64,UklGRlYAAABXRUJQVlA4IEoAAADQAQCdASoQAAwABUB8JagAApHyj5KHwAD+ddXV4lQsqTXx5RpVCGUdXQ7IfBgkuGP5IQ9bixaBuxnIE3j7wnPK98eTQYe/7GjgAA=="}
_Orientation and supports on printer bed_


## Installing

The final model in PETG is actually quite strong. There is minimal flex when inserting network cables. And yes, I realized after taking this picture that the PCIe cover above was out of alignment... it has been corrected.

![i350-t4 in case](/assets/images/posts/i350-t4-in-case.webp){: width="400" height="400" lqip="data:image/webp;base64,UklGRk4AAABXRUJQVlA4IEIAAACwAQCdASoQAAwABUB8JZQAAn+d2YKwAP6RKoRn0FJzw2dDQ8L/KpBkELUOoBTexNynFAMRASFkczdWL1LKKFJ4QAA="}
_Installed in the 2U case_


## The Files

Eventaully I'll be organizing all of my files and designs into a new GitHub repository. But for now, find it on [Thingiverse](https://www.thingiverse.com/thing:6203360)
