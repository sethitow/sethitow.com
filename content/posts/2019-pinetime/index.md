---
title: "Day One with PineTime"
date: 2019-12-24T21:46:00-08:00
draft: true
toc: false
images:
tags: 
  - pinetime
  - mbed
---
## Unboxing to Hello World

The supremely unhelpful error of 
```
InitTarget() start
InitTarget() end
TotalIRLen = ?, IRPrint = 0x..000000000000000000000000
Cannot connect to target.
```

It took me an embarisingly long time to realize I had the SWD Clock and Data lines backwards 😳

With that resolved, the J-Link connected without issue.

I tried to dump the flash but as reported by others, readback was locked. The only way forward was to erase the default applicaiton. 

```
nrfjprog --recover -f nrf52
```

https://github.com/Emeryth/sma-q2-oss
https://hackaday.io/project/85463-color-open-source-smartwatch

https://github.com/lupyuen/stm32bluepill-mynewt-sensor/tree/pinetime
https://twitter.com/MisterTechBlog/status/1196373063660519424