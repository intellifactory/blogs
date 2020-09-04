---
title: "2048 game with F# + WebSharper"
categories: "cloudsharper,f#,websharper"
abstract: "A re implementation of the popular browser game with easy to override game logic."
identity: "4034,77421"
---
This [small logic game written](http://gabrielecirulli.github.io/2048/) in JavaScript+HTML5 has spawned quite a lot of variants. There is a site called [Make Your Own 2048](http://community.usvsth3m.com/2048/) where you can switch the original's numbers to any text or image and share your creation. However, modifying the game logic requires some programming knowledge and sometimes bigger rework of the original JavaScript code.

## Try it live

[![](http://i.imgur.com/I8Mmwrm.png)](http://intellifactory.github.io/2048/)

My goal with an F#-based version was to have an easily overriden game rules class to enable writing a new variant in a few lines of code. In [CloudSharper](http://cloudsharper.com/), you can write and play your variant right in your browser window. Clone [this shared workspace](http://cloudsharper.com/clone/JankoA/6d50e34f-c0f0-4d7e-9b75-3a40f60d12e0/2048) and look for the file `Variants.fs` for editing. Build the project and open `index.html` and switch to Document/Split view for easiest testing, or use the Workspace/Deploy locally (Ctrl+D) menu option.


## Clone in CloudSharper

[![](http://i.imgur.com/BCBaI68.png)](http://cloudsharper.com/clone/JankoA/6d50e34f-c0f0-4d7e-9b75-3a40f60d12e0/2048)

Currently the grid size and the tiles' appearance can't be customized, but expect this demo to be expanded to show off more features coming to CloudSharper.
