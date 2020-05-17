---
title: PWM
weight: 1
---

## What is PWM?

PWM, short for Pulse Width Modulation, is a trick to dim a digital output by switching the output on and off quickly enough so that the average remains at a value somewhere between on and off. Using this trick, a digital output starts to behave like an analog output. One very common application is dimming of LEDs.

In the example below you can control a simulated LED using PWM. You can change the duty cycle and the speed and see how that affects the LED. Notice how, if the frequency is high enough, the output appears continuous.

<label>
  Duty cycle: <span id="pwm-dutycycle">50%</span>
  <input id="pwm-dutycycle-input" type="range" value="50" min="0" max="100"/>
</label>

<label>
  Speed: <span id="pwm-period">1000ms</span> or <span id="pwm-frequency">1.0Hz</span>
  <input id="pwm-period-input" type="range" value="0" min="0" max="30"/>
</label>

<svg viewBox="0 0 340 30" width="340" height="30">
  <rect x="0.5" y="0.5" width="302" height="28" fill="transparent" stroke="gray"/>
  <path id="pwm-path" fill="transparent" stroke="black" stroke-linecap="square" d="M 1.5 27.5 H 1.5 V 1.5 H 151.5 V 27.5 H 301.5"/>
  <circle id="pwm-led" cx="325" cy="14.5" r="10" fill="gray" stroke-width="1" stroke="gray"/>
</svg>

<style id="pwm-styles">
/* Note: these properties will be modified from JavaScript using CSSOM. */
#pwm-led {
  animation-duration: 1s;
  animation-name: pwm;
  animation-iteration-count: infinite;
}
@keyframes pwm {
  from {
    fill: transparent;
  }
  10% {
    fill: red;
  }
  50% {
    fill: red;
  }
  60% {
    fill: transparent;
  }
  to {
    fill: transparent;
  }
}
</style>

<script>
'use strict';
var period, frequency, dutyCycle;

// Update the graph and the LED animation.
function updateView() {
  if (frequency === undefined)
    return; // updateSpeed not yet called

  // Update the graph.
  let graphTop = 1.5;
  let graphBottom = 27.5;
  let graphStart = 1.5;
  let graphEnd = 301.5;
  let width = graphEnd - graphStart
  let widthTime = 1; // 1s
  let path;
  if (dutyCycle == 0) {
    // Duty cycle is 0%, draw a straight line at the bottom.
    path = 'M ' + graphStart + ' ' + graphBottom + ' H ' + graphEnd;
  } else if (dutyCycle == 1) {
    // Duty cycle is 100%, draw a straight line at the top.
    path = 'M ' + graphStart + ' ' + graphTop + ' H ' + graphEnd;
  } else {
    // Duty cycle is somewhere in between, draw a line starting at the start of
    // a pulse.
    path = 'M ' + graphStart + ' ' + graphBottom;
    for (let t=0; t<widthTime; t+=period) {
      let pulseStart = graphStart + t*width;
      path += ' H ' + pulseStart + ' V ' + graphTop;
      let pulseEnd = graphStart + (t+period*dutyCycle)*width;
      if (pulseEnd > graphEnd) {
        // End of the graph, don't move the line back.
        continue;
      }
      if (dutyCycle >= 1) {
        continue;
      }
      path += ' H ' + pulseEnd + ' V ' + graphBottom;
    }
  }
  path += ' H ' + graphEnd;
  document.querySelector('#pwm-path').setAttribute('d', path);

  // Update the LED animation CSS.
  let animation;
  let style;
  for (let rule of document.querySelector('#pwm-styles').sheet.cssRules) {
    if (rule.type == CSSRule.KEYFRAMES_RULE && rule.name == 'pwm') {
      animation = rule;
    } else if (rule.type == CSSRule.STYLE_RULE && rule.selectorText == '#pwm-led') {
      style = rule;
    }
  }
  if (dutyCycle == 1) {
    style.style.animationName = '';
    style.style.fill = 'red';
  } else if (dutyCycle == 0) {
    style.style.animationName = '';
    style.style.fill = 'transparent';
  } else if (period < 0.035) {
    // Speed is too high for most monitors/browsers to display properly.
    // This is a bit cheating, but there is no other way to make it show what
    // it is intended to show.
    style.style.animationName = '';
    style.style.fill = 'rgba(255, 0, 0, ' + dutyCycle + ')';
  } else {
    let delayPercent = (frequency/50) * 100; // soften transition in 20ms
    animation.cssRules[1].keyText = delayPercent + '%';
    animation.cssRules[2].keyText = (dutyCycle*100) + '%';
    animation.cssRules[3].keyText = (dutyCycle*100 + delayPercent) + '%';
    style.style.animationName = 'pwm';
    style.style.animationDuration = period + 's';
  }
}

// Called when the duty cycle slider is changed.
function updateDutyCycle() {
  let input = document.querySelector('#pwm-dutycycle-input');
  dutyCycle = input.value / 100;
  document.querySelector('#pwm-dutycycle').textContent = input.value + '%';
  updateView();
}

// Called when the speed slider is changed.
function updateSpeed() {
  let input = document.querySelector('#pwm-period-input');
  period = Math.pow(10, (30-input.value) / 10) / 1000;
  frequency = 1 / period;
  let periodString = (period * 1000).toFixed(0)+'ms';
  if (period < 0.01) {
    periodString = (period * 1e3).toFixed(1)+'ms';
  }
  if (period < 0.001) {
    periodString = (period * 1e6).toFixed(0)+'µs';
  }
  document.querySelector('#pwm-period').textContent = periodString;
  let frequencyString = frequency.toFixed(1) + 'Hz';
  if (frequency >= 100) {
    frequencyString = frequency.toFixed(0) + 'Hz';
  }
  document.querySelector('#pwm-frequency').textContent = frequencyString;
  updateView();
}

// Listen to slider changes.
document.querySelector('#pwm-dutycycle-input').addEventListener('input', updateDutyCycle);
document.querySelector('#pwm-period-input').addEventListener('input', updateSpeed);

// Make sure the graph and LED are initialized properly.
// The default values should work already, but Firefox keeps input state across
// refreshes so this needs to be updated anyway.
updateDutyCycle();
updateSpeed();
</script>

## How does a PWM peripheral work?

In most cases, a PWM is a combination of a counter that counts up or down and a few channels with a value that this counter is continuously compared against.

The below example is a very simple 3-bit PWM peripheral. It can only count up to 7 before it starts again at zero. Most PWM peripherals are 8, 16 or 24 bits and thus have a much larger range. You can control the 'top' value as in most real-world PWM peripherals. It also has few channels: most real-world PWMs have around four output channels.

<label>
  Counter top: <span id="timer-top">7</span>
  <input type="range" id="timer-top-input" value="7" min="0" max="7" oninput="updateTimerTop()"/>
</label>

<label>
  Channel compare A: <span class="timer-ch">5</span>
  <input type="range" class="timer-ch-input" value="5" min="0" max="7" oninput="updateChannelCompare()"/>
</label>

<label>
  Channel compare B: <span class="timer-ch">2</span>
  <input type="range" class="timer-ch-input" value="2" min="0" max="7" oninput="updateChannelCompare()"/>
</label>

<svg viewBox="0 0 250 50" width="250" height="50" style="background: #eee">
  <line x1="49.5" y1="0.5" x2="49.5" y2="49.5" fill="transparent" stroke="gray" stroke-linecap="square"/>
  <line x1="0.5" y1="49.5" x2="249.5" y2="49.5" fill="transparent" stroke="gray" stroke-linecap="square"/>
  <line id="timer-counter-bar-top" x1="28.5" y1="20.5" x2="39.5" y2="20.5" stroke="red" stroke-width="1" stroke-linecap="square"/>
  <line id="timer-counter-bar" x1="38.5" y1="20.5" x2="49.5" y2="20.5" stroke="black" stroke-width="1" stroke-linecap="square"/>
  <path id="timer-counter" stroke-width="1" fill="transparent" stroke="black"/>
</svg>

<script>
'use strict';

var timerTop;
var timerMax = 7;
var channels = [5, 2];
var counter = 7; // wrap around on first step
var counterHistory = [];
var counterAnimation;

function updateTimerTop() {
  timerTop = +document.querySelector('#timer-top-input').value;
  document.querySelector('#timer-top').textContent = timerTop;

  let counterBarTop = document.querySelector('#timer-counter-bar-top');
  counterBarTop.setAttribute('y1', 48-timerTop*6);
  counterBarTop.setAttribute('y2', 48-timerTop*6);
}

function updateChannelCompare() {
  let inputs = document.querySelectorAll('.timer-ch-input');
  let spans = document.querySelectorAll('.timer-ch');
  for (let i=0; i<inputs.length; i++) {
    channels[i] = inputs[i].value;
    spans[i].textContent = inputs[i].value;
  }
}

function doCounterStep() {
  if (counter == timerTop || counter == timerMax) {
    counter = 0;
  } else {
    counter += 1;
  }

  let counterBottom = 48;
  let counterDistance = 6;
  let counterStart = 50;

  let counterBar = document.querySelector('#timer-counter-bar');
  counterBar.setAttribute('y1', counterBottom-counter*counterDistance);
  counterBar.setAttribute('y2', counterBottom-counter*counterDistance);

  // Draw new path, including the history.
  counterHistory.unshift(counter);
  if (counterHistory.length > 22)
    counterHistory.pop();
  let path = '';
  for (let i=0; i<counterHistory.length; i++) {
    path += ' M ' + (counterStart + i*10-10) + ' ' + (counterBottom - counterHistory[i]*counterDistance) + ' h 10';
  }
  let counterEl = document.querySelector('#timer-counter');
  counterEl.setAttribute('d', path);
  if (counterAnimation) {
    counterAnimation.cancel();
  }
  counterAnimation = counterEl.animate([{transform: 'translateX(0px)'}, {transform: 'translateX(20px)'}], 666);

  setTimeout(function() {
    requestAnimationFrame(doCounterStep);
  }, 333);
}

// Update values on load.
updateChannelCompare();
updateTimerTop();

// Start animation.
doCounterStep();
</script>

While playing around with this, you may notice a few things:

  * The counter top determines the period of the output signal. It is easy to calculate the frequency from that, which is the inverse of the period.
  * The channel compare value determines the duty cycle of the signal. Every time the counter matches zero, all channel outputs are turned on and every time a channel matches the compare value, the channel is turned off again.
  * While you can change the period (and thus the frequency) of the output signal by changing the top value, the channel compare values will not automatically change with it and thus the duty cycle will change if you change the frequency: even though the 'on' period remains the same, the 'off' period changes.
  * While you can control the duty cycle of every channel independently, you cannot control the period (and thus the frequency) for individual channels. You can only change it for the entire PWM peripheral.

There are also a few things that are supported by many real-world PWMs but not by this simplified example:

  * (Nearly?) all PWM peripherls support a prescaler to reduce the tick frequency, which allows for much longer periods and thus much lower frequencies than would otherwise be possible.
  * Most PWM peripherals are actually called timer/counter peripherals, because they can also count events (by incrementing the counter with some external trigger instead of from an internal clock source) and they allow timestamping events by loading the current counter into one of the channels. These channels are also called capture/compare channels because they cannot just compare the value to the current counter but can also be loaded with the current counter.
  * PWM peripherals support many more configurations than just counting up. Some count down by default (reloading with the top value when they hit zero) and some count both up and down in a kind of triangle waveform. You don't need all of that for just controlling LEDs, but some hardware needs very precise control over the generated waveform.
  * These peripherals are also called timers, because sometimes they can be used as just that: a timer, with no external connections. You can keep track of the current time by reading the counter value and registering an interrupt on overflow so the timer value keeps incrementing.
  * Some chips (such as many AVRs) do not have a dedicated top value. Instead, you can assign one of the capture/compare channels as the top value. Of course, that means you cannot use that channel for generating a PWM output.

## Using PWM in TinyGo

TinyGo abstracts away most chip specific details in a relatively simple PWM interface. For example, if you know that you can use the LED pin in PWM 0, you can easily change the brightness:

```go
import "machine"

func main() {
    // Configure the PWM peripheral. The default PWMConfig is fine for many
    // purposes.
    pwm := machine.PWM0
    err := pwm.Configure(machine.PWMConfig{})
    handleError(err, "failed to configure PWM0")

    // Obtain a PWM channel.
    ch, err := pwm.Channel(machine.LED)
    handleError(err, "failed to obtain PWM channel for LED")

    // Set the channel to a 25% duty cycle.
    ch.Set(pwm.Top() / 4)

    // Wait before exiting.
    time.Sleep(time.Hour)
}
```

You can also control multiple LEDs from a single PWM peripheral, as long as they are supported by that PWM peripheral:

```go
import "machine"

func main() {
    // Configure the PWM peripheral. The default PWMConfig is fine for many
    // purposes.
    pwm := machine.PWM0
    err := pwm.Configure(machine.PWMConfig{})
    handleError(err, "failed to configure PWM0")

    // Set the first LED to 25% brightness, as before.
    ch1, err := pwm.Channel(machine.LED1)
    handleError(err, "failed to obtain PWM channel for LED1")
    ch1.Set(pwm.Top() / 4)

    // Set the second LED to 50% brightness.
    ch2, err := pwm.Channel(machine.LED2)
    handleError(err, "failed to obtain PWM channel for LED2")
    ch2.Set(pwm.Top() / 2)

    // Set the third LED to 75% brightness.
    ch3, err := pwm.Channel(machine.LED3)
    handleError(err, "failed to obtain PWM channel for LED3")
    ch3.Set(pwm.Top() * 3 / 4)

    // Wait before exiting.
    time.Sleep(time.Hour)
}
```

You can find which PWM peripheral you can use for any given pin in the board documentation. There it may also indicate a channel number. You can often ignore the channel number, but if you use two different pins with the same PWM peripheral and the same channel number, the two pins will be linked together (you cannot set the duty cycle of one without affecting the other).

### Controlling a servo

The example below controls a typical servo, that uses 20ms between pulses and a pulse width of 1.5ms for the neutral state:

```go
import "machine"

func main() {
    // Configure the PWM peripheral with a period of 20ms.
    pwm := machine.PWM0
    err := pwm.Configure(machine.PWMConfig{
        // The period must be 20ms.
        // The period is specified in nanoseconds, so we actually specify
        // 20_000_000ns, or 20e6 (using scientific notation).
        Period: 20e6,
    })
    handleError(err, "failed to configure PWM0")

    // Obtain a PWM channel.
    ch, err := pwm.Channel(machine.D4)
    handleError(err, "failed to obtain PWM channel for D4")

    // To set the servo to the neutral state, we need to send pulses that are
    // 1.5ms wide. Because the entire pulse is 20ms wide, we use the following
    // formula to get a 1.5ms wide pulse:
    //     1.5ms = 20ms * 1.5 / 20
    //     1.5ms = 20ms * 3   / 40
    ch.Set(pwm.Top() * 3 / 40)

    // Wait before exiting.
    time.Sleep(time.Hour)
}
```


### Controlling a piezo buzzer

For a piezo buzzer it's important to get the frequency exactly right, otherwise it'll sound wrong. Luckily, this is possible in the PWM API.

```go
import "machine"

func main() {
    // Configure the PWM peripheral with a specific frequency.
    pwm := machine.PWM0
    err := pwm.Configure(machine.PWMConfig{
        // The frequency for the A4 tone is 440Hz.
        // We can't specify the frequency in Hertz, however. We need to use the
        // period (in nanoseconds) instead. The formula below is actually the
        // following formula:
        //     period = 1s / 440Hz
        // In nanoseconds:
        //     period = 1_000_000_000ns / 440Hz
        Period: 1e9 / 440,
    })
    handleError(err, "failed to configure PWM0")

    // Obtain a PWM channel from a pin connected to a piezo buzzer.
    ch, err := pwm.Channel(machine.D10)
    handleError(err, "failed to obtain PWM channel for D10")

    for {
        // Make noise: set the channel to a 50% duty cycle and wait a bit.
        ch.Set(pwm.Top() / 2)
        time.Sleep(500 * time.Millisecond)

        // Stop the noise: set the channel to a 0% duty cycle (off) and wait a
        // bit.
        ch.Set(0)
        time.Sleep(500 * time.Millisecond)
    }
}
```

Of course, you can't mix servos and piezo buzzers on the same PWM peripheral as you can only set a single frequency per PWM peripheral.
