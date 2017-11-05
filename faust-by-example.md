# Faust by Example

## Generating sounds
mono white noise, volume at 0.1
```
import("music.lib");

process = noise * 0.1;
```

mono white noise, slider for volume
```
import("music.lib");

process = noise * vslider("volume", 0.1, 0, 1, 0.01);
```

assigning noise to a variable
```
import("music.lib");

myNoise = noise * vslider("volume", 0.1, 0, 1, 0.01);

process = myNoise;
```

stereo white noise
```
import("music.lib");

myNoise = noise * vslider("volume", 0.1, 0, 1, 0.01);

process = myNoise, myNoise;
```

stereo white noise, seperate sliders
```
import("music.lib");

leftNoise = noise * vslider("left volume", 0.1, 0, 1, 0.01);
rightNoise = noise * vslider("right volume", 0.1, 0, 1, 0.01);

process = leftNoise, rightNoise;
```

## Audio input
```
input = _;
process = input;
```

Delay, using the `@` operator, delays by number of samples
```
input = _;
delayed = input @ hslider("delay", 0, 0, 44100, 1);
process = input + delayed;
```

## Maths operators

`+ - * /` do what you would expect
`%` is modulo
`^` is power

### Comparison operations

Usual friends `< <= == > >= !=` result is 1 for true, 0 for false.

Noise gate:
```
import("stdfaust.lib");

signal = _;
threshold = 0.1;
attack = 0.01;
release = 0.25;

gate = signal > threshold : si.lag_ud(attack, release);

process = signal * gate;
```


### bitwise operators are
`| & xor << >>`



### Syntactic sugar

These are all syntactic sugar for function calls, so you can do

```
+(1, 2);
^(3, 2);
&(1, 0);
<(2, 5);
```

### Unary operators

Warning! Limited use of these, they can only be used with numbers and variables

These are ok
```
-0.5;
x = 20;
-x;
```

These are not
```
-(1.0 / 2.0);
+(5.0 + 10.0)
```

Because these are partially complete functions (partial applications), lambdas that have not been called yet.

```
import ("music.lib");

x = -0.01;  // number
y = +(x);   // partial application, a function that is not fully called
z = y(0);   // call the function, get the value

process = noise * z;
```

## functions

convenience syntax
```
quiet(signal) = signal * 0.1;
```

full syntax
```
quiet = \(signal).(signal * 0.1);
```

operators are functions
```
// a and b are equivalent
a = 1 * 0.5;
b = *(1, 0.5);
```

## The five types of composition

### Parallel

`,` puts two blocks in parallel, so two mono signals in parallel become a stereo signal

### Sequential

`:` is for sequences, so `guitar : distortion` means that the output of the `guitar` signal is the input of the `distortion` signal.

### Split

`<:` will split the input signal, so `guitar <: stereoReverb` will split the mono `guitar` signal into two channels to match the two inputs of `stereoReverb`. The number of inputs of the second signal must be a multiple of the outputs of the first signal, so you can plug a mono (1) guitar into a stereo (2) reverb, but not a stereo (2) reverb into a surround sound (5) mixer.

These will wrap, so if you have a 2 channel reverb going into a 4 channel mixer then the mixer's inputs will look like:

```
mixerIn1 = reverbOut1
mixerIn2 = reverbOut2
mixerIn3 = reverbOut1
mixerIn4 = reverbOut2

```

#### Selecting a channel

If you need to just get a single channel from a multi channel signal use `selector` if the channel is known in advance and won't change (compile time) or `selectn` if you want to change the channel while playing (run-time).

```
import("music.lib");

monoNoise = noise * 0.1;
stereoNoise = monoNoise, monoNoise;
rightChannel = stereoNoise : selector(0, outputs(stereoNoise));

process = rightChannel;
```

Alternatively you can use the blocking operator

```
left = _ , _ : _ , !;
```

This can be useful if you want to split a multichannel signal into separate variables

```
left  = _ , _ : _ , !;
right = _ , _ : ! , _;
```



### Merge

`:>` is the buddy of split, so `stereoReverb :> monoAmplifier` will turn the two channels of the `stereoReverb` into a mono signal to match the `monoAmplifier` input. Like split, the number of channels must match or be a multiple, so while a stereo (2) channel reverb can go into a mono (1) channel amplifier, a surround sound (5) mixer cannot go into a stereo (2) reverb. The channels also wrap in the same way as Split.

### Recursion

`~ ` allows blocks to connect to each other in a loop, in the way that a delay effect has a feedback control, so the wet delay effect can be itself delayed, creating a tail of delays.

Imagine you have a stereo mixer that has two inputs and one output, you could connect your guitar to channel 1, and then the output to your amplifier.
Now if you put a little digital splitter box with a volume control between the mixer and the amplifier, you could send an exact copy of the mixer's output back into channel 2 of the mixer. To do the splitting, the box needs a bit of time, which is exactly 1 sample (1 / 44100 at CD quality). So now what you get out of the mixer, is the original guitar signal *and* the delayed guitar signal. As anyone who has messed around with delay effects knows, this would quickly lead to very loud feedback, hence the volume control that allows us to reduce the volume of the delayed guitar signal before adding it to the mixer.

In code this would look like this
```
feedbackVolume = 0.1;
+(guitar) ~ *(feedbackVolume);
```

Our mixer in this case is the `+` operator, which adds our two signals together.

## Inputs and outputs

We can get the number of inputs and outputs of an expression, so `outputs(monoSignal)` would return 1, `outputs(stereoSignal)` would return 2, and `inputs(+)` would return 2.

## Iterations

`sum` will give the sum of the expressions

```
sum(i, 3, i / 100.0);
```

Will be 0.1 + 0.01 + 0.02 = 0.03

`prod` gives the product

```
prod(i, 3, (i + 1) / 10.0);
```

Is 0.1 * 0.2 * 0.3 = 0.006

`par` can be used to create multiple parallel signals

```
import ("music.lib");

myNoise = noise * 0.01;
stereo = myNoise, myNoise;
stereo2 = par(i, 2, myNoise);

process = stereo2;
```

`seq` creates a sequence of expressions, which must take input

```
mySeq = seq(i, 3, /(2));

mySeq(0.5);
```

This defines mySeq, then calls it giving 0.5 as input, the result is

(((0.5 / 2) / 2) / 2) = 0.0625

or

0.5 / 2 = 0.25
0.25 / 2 = 0.125
0.125 / 2 = 0.0625

## Environments

Environments are similar to environments in SuperCollider, or objects in JavaScript. They are created with a literal syntax, and accessed with dot notation.

```
import ("music.lib");

myEnv = environment {
  sig = noise;
  mul = 0.1;
};

process = myEnv.sig * myEnv.mul;
```

### Loading from a file

Environments are loaded from a file like so

```
myEnv = library("myLibrary.lib");
process = myEnv.sig * myEnv.mul;
```
## Writing to a file

compile using

`faust2sndfile patch.dsp`

Which will generate a new executable, called `patch`

If your faust patch generates audio, and doesn't process input you will need to generate a silent audio for the duration you want. (Using a DAW, OcenAudio, Audacity etc)
Then run

`patch input.wav output.wav`



`
