---
title: "Microphone Amplifier for Introverts"
date: 2019-08-23T17:56:46+02:00
---
For whom?
---------

This guide is intended for introverts but it actually works for everybody who
has difficulties speaking loudly with a microphone. At the moment it is only
for Windows users but I expect I will soon find out how to do the same on
Linux. I don't own a computer on OSX so if someone wants to add their tutorial
here, feel free to do send me an email.

You must probably want to ask me: why not just turn the microphone sensitivity
higher. The problem is that it would capture more surrounding sounds, and those
sounds will be louder (depending on your microphone) than the solution proposed
here.

Unfortunately I couldn't find a way to make sensible spectrogram to show the
actual difference but I heavily tested this and people do hear a difference. I
personally do feel much more comfortable.

If you don't play video games, you could probably simply put your mouth closer
to the microphone while reducing the sensitivity and it will do the exact same
thing.

I'm proposing 2 solutions in this post. The first one is a program to install,
easy to use. Entirely graphical. It's still better to be an advanced user to do
it. The second one gives you a lot more flexibility but you will need to use
the command line and do some trial and error. It's quite straightforward but
you need to be comfortable with the command line.

*Warning:* always use a headset if you want to avoid horrible echos during your
tests.


(1) The easy solution
---------------------

Go on [https://www.vb-audio.com/](https://www.vb-audio.com/) and install the
[Voicemeeter](https://www.vb-audio.com/Voicemeeter/index.htm) (under "Audio
Apps").

You will need to reboot your machine (even on Windows 10) and it will create 2
new audio devices for you. One is a microphone and one is an audio output.

The program needs to be running for this solution to work. You might want to
add it to your startup applications or start it manually whenever you need to
use the microphone.

There are 2 interesting panels in this program:
1.  The first one on the left: microphone input / hardware input
2.  The last one on the right: audio device output / hardware out

On the first panel there is a squared button "1". Click on it and select the
microphone you want to listen from. On the last panel there is a squared button
"A1", click on it and select the audio device you want to use to test the
amplification.

When both are selected you can speak to your microphone and you will hear your
voice directly. Now you can change the slider "Fader Gain" on the first panel
to amplify the volume. Unfortunately there is a limit of +12dB but hopefully
that will be enough for you.

When you are satisfied with the amplification, mute the output (4th panel) by
clicking the "M" button. Then start using any program you want. You will
probably need to change the microphone device used by the application using the
Windows control panel and select the "Voicemeeter" one. There is a panel "App
volume and device preferences" somewhere in the control panel of Windows.

(2) The complete solution
-------------------------

If you have installed the Voicemeeter of the previous tutorial, uninstall it
and reboot.

Go on [https://www.vb-audio.com/](https://www.vb-audio.com/) and install the
[Virtual Audio Cable](https://www.vb-audio.com/Cable/index.htm) (under "Audio
Apps"). Reboot (you must).

Go on [http://sox.sourceforge.net/](http://sox.sourceforge.net/) and install
[SoX for Windows](https://sourceforge.net/projects/sox/files/sox/).

Open the command line, go to the directory and use this command to stream
without modification the sound of your microphone to your headset:

```dos
sox.exe -t waveaudio "Microphone (USB PnP Audio Device)" -t waveaudio "Speakers (2- Schiit Modi 3)"
```

This is an example, you will need to replace the name of the microphone by your
own microphone and the name of your headset by your own headset.

Now we can add filters at the end of the command line to downmix the channels
to 1 channel (it improves a bit the volume) and increase the volume:

```dos
sox.exe -t waveaudio "..." -t waveaudio "..." channels 1 vol 30db
```

30dB is already a lot. You should hear a big difference. If the sound
saturates, you need to decrease this value.

When you are satisfied with the result, change the output audio device to
"CABLE Input (VB-Audio Virtual Cable)" (the name might be slightly different
but you should be able to find it easily) and the command will look like this:

```dos
sox.exe -t waveaudio "Microphone (...)" -t waveaudio "CABLE Input (VB-Audio Virtual Cable)" channels 1 vol 30db
```

You need this command to be running for the solution to work. You can now start
using the program you want. You will probably need to change the microphone
input of the program to the "CABLE Output" microphone using the Windows'
control panel "App volume and device preferences".

### Noise reduction

To improve the quality of the sound you can add a noise reduction effect. To do
that, you will first need to create a noise profile. Make sure your room is
completely silent, make no noise at all, and start the profiling with this
command:

```dos
sox.exe -t waveaudio "Microphone (...)" -n channels 1 vol 30 noiseprof noise.prof
```

Press Ctrl-C to stop the recording. A file `noise.prof` should appear in the
current working directory. You can now test the noise reduction with this
command:

```dos
sox.exe -t waveaudio "Microphone (...)" -t waveaudio "Speakers (...)" channels 1 vol 30db noisered noise.prof 0.3
```

You will need to test different values than 0.3. The maximum is 1 but you
should select a value between 0.1 and 0.5. The expected result is that you hear
no sound at all when you make no sound at all.

If, like me, the weather is super hot and you need to have a fan running in
your office, you might want to keep it running while you are running the noise
profiling so it also gets reduced.

Too much noise reduction leads to sound distortion (you will hear a cyborg
voice).
