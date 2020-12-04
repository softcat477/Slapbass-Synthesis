# Slapbass Synthesis
A final project for MUMT618.
## Outline
* [Objective](#Objective)
* [Slap Technique](#Slap-Technique)
* [Model Implementation](#Model-Implementation)
* [Simulation Results](#Simulation-Results)  
* [Combine with musicXML parser](#Combine-with-musicXML-parser)
* [Conclusion](#Conclusion)
* [Reference](#Reference)

## Objective
The objective of this project is to synthesize the slap technique of an electric bass guitar and use the model to synthesize slap bass lines from `.musicXml` files to let users take advantage of music notation software to edit the score.

## Slap Technique
Slapping is a way to produce sounds on string instruments and is commonly used in funk and R&B music. The timbre sounds more percussive and metallic than plucking the string with fingertips.

The slapping technique contains two basic types: slapping and popping. In slapping, the player strikes the string with the knuckle of the thumb and rest the thumb on the next string or move the thumb away from the string; in popping, the player uses the index finger to pull the string away from the fretboard and release the string. In both types, the string snaps back to the fingerboard and hits the fret to create the metallic and percussive sound.

{%youtube 6rxZy5gJl7Q %}
<span style="font-family:Didot; font-size:14px">↑ A slap bassine. Note the timbre and groove difference between the normal plucking technique at 00:22~00:24 and the slapping+popping technique at 00:25~00:30.</span>

## Model Implementation
## Model Implementation
The implementation synthesizes the slapping technique with digital waveguide models proposed in [A waveguide model for slapbass synthesis](https://ieeexplore.ieee.org/document/599670) and [Methods for simulating string collisions with rigid spatial obstacles](https://ieeexplore.ieee.org/document/1285874). Implementation is developed with the [The Synthesis ToolKit](https://ccrma.stanford.edu/software/stk/index.html) in C++.

To simulate a vibrating string colliding with frets, the model will encounter three conditions:
1. String vibrates without collision.
2. String collides with frets.
3. String leaves the fret.
### String vibrates without collision
In this condition, the output signal is synthesized with a waveguide model that consists of 2 delay lines, 2 reflectance filters, and a pickup circuit.
#### Delay lines
Delay lines are used to simulate the string displacement. One for the positive wave moving to the right, the other for the negative wave moving to the left. 

Delay lines are implemented with `stk::DelayL`, with the length set to `sr/(2*f0)`, where `sr` is the sampling rate and `f0` is the fundamental frequency.

Though delay lines simulate the string displacement for slap and pop, delay lines are initialized in different ways. When synthesizing with the popping technique, since the string is pulled away from the fretboard in popping, the initialization of delay lines is set to a triangular shape where the top corresponds to the pluck position.
![](https://i.imgur.com/iVHyT5x.png)
<span style="font-family:Didot; font-size:14px">↑ Figure 3-1: The initial string displacement for popping. Green: Positive delay line. Blue: Negative delay line. Red: Total displacement.</span>

For the slapping technique, the string is struck, and it is more natural to initialize delay lines with a velocity pulse to simulate the string velocity. However, because we use the fret height to test if the string collides with the fret, delay lines are restricted to simulate the string displacement only. Therefore we initialize delay lines by converting the initial velocity to the equivalent initial string displacement.
![](https://i.imgur.com/KvrrHno.png)
<span style="font-family:Didot; font-size:14px">↑ Figure 3-2: The initial string displacement for slapping. Green: Positive delay line. Blue: Negative delay line. Red: Total displacement.</span>

The corners in the initial displacement are rounded with a quadratic Bézier curve

$\boldsymbol{B(t)} = (1-t)^2\boldsymbol{P_0}+2t(1-t)\boldsymbol{P_1}+t^2\boldsymbol{P_2}, 0 \leq t \leq 1$

To round a corner, $P1$ is set to `(c, y[c])`, where `c` is the delay line index of the corner, `y[c]` is the valued stored in `c`th delay unit in the delay line `y`; $P0$ is set to `(c-2, y[c-2])`, which is 2 delay units to the left of the corner; $P2$ is set to `(c+2, y[c+2])`, which is 2 delay units to the right of the corner. Then we infer 5 points from the Bézier curve with $t=[0.00, 0.25, 0.50, 0.75, 1.00]$ and assign them to `y[c-2]~y[c+2]` accordingly to get the rounded corner.
![](https://i.imgur.com/OWzTjEI.png)
<span style="font-family:Didot; font-size:14px">↑ Figure 3-3: Comparison of a corner and the corner rounded with a quadratic Bézier curve by setting $P0=(0, 0.0), P1=(2, 1.0), P2=(4, 0.0)$.</span>

#### Reflectance filters
There are two reflection filters at the body-end and the head-end of the string. For the body-end of the string, we adopt an ideal reflectance filter with transfer function $R_L(z)=-1$ to send the output from the negative delay line back to the input of the positive delay line with a sign inversion; for the head-end of the string, the output of the positive delay line is passed through a single pow low pass filter with transfer function $R_R(z) = \frac {0.4} {1-0.6z^-1}$, and the result is pushed into the negative delay line.

The ideal reflectance filter is implemented by multiplying the output from the negative delay line with -1 and pushing it into the positive delay line. The single-pole low pass filter is implemented with `stk::OnePole` with `a1=-0.6, b0=0.4`. 

#### Pickup circuit
The pickup circuit is used to simulate the magnetic pickup of an electric bass guitar. A magnetic pickup produces current when the vibrating string changes the magnetic field. The pickup circuit can be approximated with a first-order FIR filter with transfer function $P_1(z)=1-z^{-1}$ to get the string velocity from string displacement.

The output signal from the pickup circuit is further sent to a single pow low pass filter $P_2(z) = \frac {0.2} {1-0.8z^-1}$ to attenuate the high frequency components in the signal.

The first order FIR filter is implemented by subtracting the total displacement of the last frame from the total displacement of this frame; the low pass filter is implemented with `stk::OnePole` with `a1=-0.8, b0=0.2`.

#### Test if the string collides
In each frame, we need to know the string touches the fret or not with the inequality test:

$y^+(x_f, t)+y^-(x_f, t)<y_{f}$, where $x_f$ is the delay unit index of the fret position, $y_f$ is the fret height and $t$ is the frame index.

The result is `True` when the total displacement $y^+(x_f, t)+y^-(x_f, t)$ is less than the fret height $y_f$ and implies that the string collides with the fret; the result yields `False` when the string moves without collision.

#### Simulation Diagram
`Fig 3-4` shows the simulation diagram when simulating a free moving string.
![](https://i.imgur.com/2NQHNJQ.jpg)
<span style="font-family:Didot; font-size:14px">↑ Figure 3-4: The diagram of a waveguide model to simulate a free moving string.</span>

In each frame, we do the following steps:
1. Do the inequality test.
2. Update delay lines according to the test result.

### String collides with the fret.
To simulate a string collides with the fret(pass the inequality test), we split the merged model into two disconnected models. One for the string to the right of the fret position, the other for the string to the left.

### Delay lines
We need four delay lines for two waveguide models. The left model and the right model overlap for a delay unit, which means 
$length(y^+)=length(y^+_L)+length(y^+_R)-1$ and
$length(y^-)=length(y^+_L)+length(y^-_R)-1$.
![](https://i.imgur.com/CSJneCW.jpg)
<span style="font-family:Didot; font-size:14px">↑ Figure 3-5: An illustration of the overlapped delay unit. $y^+$ and $y^-$ are the positive and negative delay lines of the merged waveguide model for the free-moving string. $y^+_L$ and $y^-_L$ are delay lines of the left model. $y^+_R$ and $y^-_R$ are delay lines of the right model</span>

New delay lines are initialized with values copied from the merged model's delay line. The output signal is synthesized from the model that contains the pickup circuit.

### Offset
Offset $C_{off}$ at frame $t_n$ is the difference between the positive delay lines at the overlapped delay unit in two models. 

$C_{off}(t_n)=y^+_L(x_{f}, t_{n-1})-y^+_{R}(x_{f}, t_{n-1})$

If we do not remove the offset, discontinuities occur at the fret position in both positive wave displacement and negative wave displacement, and the string incorrectly sticks on the fret for too long. To avoid the incorrect simulation, we remove the offset from the right model in each frame with equations:

$y^+_R=y^+_R+C_{off}$
$y^-_R=y^-_R-C_{off}$

### Reflectance filters
We keep the reflectance filter $R_L$ and $R_R$ of the merged model and use them in the left model and right model. As to the reflectance around the fret position, the output of a delay line is subtracted from the fret height, and the result is pushed into the other delay line.

#### Simulation Diagram
`Fig 3-6` shows the simulation diagram when the string collides with the fret.
![](https://i.imgur.com/X410SC8.jpg)
<span style="font-family:Didot; font-size:14px">↑ Figure 3-6: The diagram of a waveguide model to simulate a string collides with a fret.</span>

In each frame, do the following steps:
1. Find the offset.
2. Remove the offset from the right model.
3. Do the inequality test.
4. Update delay lines according to the test result.

## The string leaves the fret.
The string leaves the fret when the inequality test yields a `False` result. Then we merge the two models back to the merged model. We take the overlapped delay unit of the left model and discard the overlapped delay unit of the right model.

The values of delay units in the left model and right model are copied and pasted to the merged model and the output signal is synthesized with the merged model.

#### Simulation Diagram
The diagram is the same as the case of simulating a free-moving string in `Fig 3-4`.

In each frame, we do the following steps:
1. Do the inequality test.
2. Update delay lines according to the test result.

## Simulation Results

![](https://i.imgur.com/WrvWnuW.png)
<span style="font-family:Didot; font-size:14px">↑ Figure 4-1: Wave displacement with the offset. Blue: Right traveling wave; Green: Left traveling wave; Green: Wave displacement; Blue vertical line: Fret height. Note the discontinuities in the positive delay line and negative delay line at the fret position.</span>

![](https://i.imgur.com/erj4WQh.png)
<span style="font-family:Didot; font-size:14px">↑ Figure 4-2: Wave displacement without the offset. Discontinuities are removed.</span>

`Fig 4-1` and `Fig 4-2` shows the string displacement with and without the offset. The simulation has the following configuration: 
```
f0 = 98.0hz;
fret height = -0.25;
fret position = 0.23*delay line length;
pickup position = 0.14*delay line length;
pickup position = 0.14*delay line length;
```
In `Fig 4-1`, discontinuities occur in the left and right traveling waves at the fret position. In `Fig 4-2`, the discontinuities are removed when the offset is removed from each frame.

![](https://i.imgur.com/cqDN47a.png)
<span style="font-family:Didot; font-size:14px">↑ Figure 4-3: The pickup signal of a popped G2 with the offset. [Sound](https://soundcloud.com/user-551151460/pop-g2-with-offset?in=user-551151460/sets/slap-bass-synthesis)</span>
![](https://i.imgur.com/jb7QiLQ.png)
<span style="font-family:Didot; font-size:14px">↑ Figure 4-4: The pickup signal of a popped G2 without the offset. [Sound](https://soundcloud.com/user-551151460/pop-g2-no-offset)</span>

The output signal without the offset makes more buzzing sounds caused by the collision between the string and the fret. The number of frames that the string contacts the fret is 568 frames in `Fig 4-4` and 1197 frames in `Fig 4-4`. The string contacts with the fret more often when removing the offset, which might be the reason for the buzzing sounds.

![](https://i.imgur.com/3jJzrLB.png)
<span style="font-family:Didot; font-size:14px">↑ Figure 4-5: The pickup signal of a slapped G2 without the offset. [Sound](https://soundcloud.com/user-551151460/slap-g2-no-offset?in=user-551151460/sets/slap-bass-synthesis)</span>
![](https://i.imgur.com/X17YMcx.png)
<span style="font-family:Didot; font-size:14px">↑ Figure 4-6: The pickup signal of a plucked G2. [Sound](https://soundcloud.com/user-551151460/pluck-g2?in=user-551151460/sets/slap-bass-synthesis)</span>

The result of a slapped note in `Fig 4-5` sounds less percussive than a popped note,  but it still sounds more percussive than a plucked note in `Fig 4-6`, which is the same as the timbre difference between slapping, popping, and plucking on a bass guitar. Note that the timbre of the synthesized result with popped(`Fig 4-4`), slapped(`Fig 4-5`), and plucked(`Fig 4-6`) are different.

## Combine with musicXML parser
![](https://i.imgur.com/QnfAKji.jpg)
<span style="font-family:Didot; font-size:14px">↑ Figure 5-1: An illustration of the musicXML parser. [Sound](https://soundcloud.com/user-551151460/bassline?in=user-551151460/sets/slap-bass-synthesis)</span>

The implementation can synthesize slap bass lines from `musicXML` files with a musicXML parser built in Python. First, users edit the bass line with any music notation software and annotate the playing technique of each note with `T` for slapping and `P` for popping as lyrics. After editing, the bass line is exported to a `.musicXML` file. Next, the musicXML parser parses the file and get the fundamental frequency and duration of each note and rest in the bass line. Finally, the parser invokes the model to synthesize the slap bass line. 

We use [pybind11](https://www.google.com/search?client=firefox-b-d&q=pybind11) to invoke C++ functions in the parser. To synthesize a slap bass line, we synthesize each note with the model and append results to `stk::FileWvOut` to combine notes into a bass line.

## Conclusion
This project implements the waveguide model to synthesize the slap technique. The model corrects the discontinuities and the simulation results show differences between playing techniques. The model is combined with a musicXML parser to synthesis bass lines from musicXML files. For now, the project only considers one fret. In future work, more frets can be added to synthesize realistic sounds.

## Reference
E.Rank and G.Kubin,“A waveguide model for slapbass synthesis,” in Pmc IEEE ICASSP, 1997.

A. Krishnaswamy and J. O. Smith, “Methods for simulating string collisions with rigid spatial obstacles,” inProc. IEEE Worhshop Applicat.Signal Process. Audio Acoust., New Paltz, NY, 2003, pp. 213–220.
