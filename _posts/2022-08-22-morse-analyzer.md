---
title: "Morse Analyzer Released"
layout: post
categories: [morse, signal-processing]
customexcerpt: "Hobby project to decode Morse codes from audio files."
---

Always wanted to learn Morse code. I don't know why but I could not learn it. So, I wrote a small program to meet my
needs.

First I used Jupyter Notebooks to test and learn audio processing on Python. Being able to see and change the results of
the applied effects was really helpful. I tested several methods and libraries (`librosa` was the first one I've tried.
It worked well but it was so slow I had to replace it with `scipy`). After some experimenting I have found the necessary
aspects to extract Morse from the audio file.

The problem with the Jupyter Notebooks is to navigating between cells and changing the input values of the functions.
`ipywidgets` package helped the latter but the first issue remained. To extract Morse I had to test several
combinations of filters. So, I decided to write a GUI. It needs to be simple, because I do not want to waste unnecessary
time on this little project.

After some research I came upon `PySimpleGUI`. Simple, very simple library to use and learn. This was my first GUI
application on Python and it was very easy to implement what I have in my mind.

Here is the screen shot of the application:

<div class="img-wrapper no-scroll" markdown="block">

![ss](https://github.com/spaceymonk/morse-analyzer/raw/master/ss.jpg)

</div>

And here is the [Video Tutorial](https://youtu.be/KeBg3VXJFjk).

You can find the source code on [GitHub repository](https://github.com/spaceymonk/morse-analyzer).

------

After I published my work on GitHub the author of the PySimpleGUI created an
[issue](https://github.com/spaceymonk/morse-analyzer/issues/1) on the repo. That was an ... unexpected compliment! I
want to thank him again. What a coder!

