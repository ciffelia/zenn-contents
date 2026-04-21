---
title: Raspberry PiのLEDを緑色だけ点灯させる
emoji: 💡
type: tech
topics:
  - raspberry-pi
published: true
---

```ini:/boot/config.txt
# Turn off PWR LED (Red)
dtparam=pwr_led_trigger=default-on
dtparam=pwr_led_activelow=off

# Turn on ACT LED (Green)
dtparam=act_led_trigger=default-on
```
