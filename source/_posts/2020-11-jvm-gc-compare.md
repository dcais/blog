---
title: 垃圾收集比较
date: 2020-11-12 10:03:01
tags: jvm gc
---

|                   | YoungGC/MinorGc | OldGC/MajorGC                                              |
| ----------------- | --------------- | ---------------------------------------------------------- |
| Serial            | {STW} copy      | {STW}mark&compact                                          |
| Parallel Scavenge | {STW} copy      | {STW}mark&compact                                          |
| CMS               |                 | {STW}initMark => {Conc}Mark =>{STW}FinishMark=>{Conc}Sweep |
| G1                |                 | {STW}initMark=>{Conc}Mark=>                                |
| ZGC               |                 |                                                            |

