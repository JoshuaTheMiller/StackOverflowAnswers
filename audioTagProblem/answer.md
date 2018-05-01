Try copy+pasting the following block:

    <!DOCTYPE html>
    <html>
    <body>
        <audio controls>
            <source src="audio/horse.mp3" type="audio/mpeg">
        </audio>  
        </body>
    </html>

# Short Explanation

I switched your quotes to be only `"` instead of `“` and `”`.

# Long Explanation

The only thing I can see (and verify) is that you're using some funky quotation marks...

Instead of just using a `"` (a "standard" quotation mark), you are using a `“` and a `”` (a left double quotation mark and right double quotation mark). Browsers tend to not handle left and right double quotation marks as most people might expect (it's not a bug). Because of that, your audio tag probably has an incorrect `src` being set. For example, when I use the left and right double quotation marks and then inspect the source of the audio tag, I see the following:

    <source src="â€œaudio/horse.mp3â€" type="â€œaudio/mpegâ€/">

Obviously, that's not the path you were looking for.

If you look at the unicode values for them (just copy+paste `“”"` into the search bar at [UnicodeLookup.com][lookup]), you can easily see the difference between the three types of quotes.

I've set up a test demonstrating this and you can view it [here][example]. The element on the left has its source set with standard quotes, and the one on the right has its source set with the "special" quotes.

 [lookup]: https://unicodelookup.com
 [example]: https://rawgit.com/TheFlyingCaveman/StackOverflowAnswers/master/audioTagProblem/horseAudioTest.html
