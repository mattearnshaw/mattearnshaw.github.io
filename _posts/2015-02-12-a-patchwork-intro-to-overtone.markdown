---
layout: post
title:  "A Patchwork Intro to Overtone, Part 1"
date:   2015-02-12 13:46:56
categories: overtone
---
[Overtone][overtone] is a joyous synthesis of Clojure and Supercollider.

For the most part, it can be considered a Clojure wrapper for
Supercollider. So if you're an SC expert, you know most of Overtone
already, although it rolls its own [scheduler][at-at] amongst
other things.

However, I'm not an SC expert, so most of my introduction to its
abstractions has come through my ongoing study of Overtone - I assume
the reader will be in a similar position. You do not need to learn SC
in order to use Overtone!

While the documentation is quite good, particularly the
[examples][examples], there's a lack of information in some areas.

These posts will essentially be a learning diary, and so somewhat
unstructured and not comprehensive, but I hope they will be useful. I
will not cover anything about Clojure specifically.

*******

Server
======

"Supercollider" generally refers to both its synthesis server
(`scsynth`), and the client language (`sclang`) which communicate
using OSC. Overtone is a Clojure-based client for `scsynth`.

To make sound, you need to have a server booted. The simplest way to
do this is to use the "internal" server, which is booted when you load
the overtone.live namespace.

{% highlight clojure %}
user> (require '[overtone.live :refer :all])
--> Loading Overtone...
--> Booting internal SuperCollider server...
{% endhighlight %}

overtone.live also includes most of the names from the other overtone
namespaces, excluding instruments. You may see a warning about `JVM
argument TieredStopAtLevel=1`, just add the suggested `:jvm-opts ^:replace []`
to your `project.clj` if so - supposedly it helps performance.

If things get screwed up, you can kill and reboot the internal server.

{% highlight clojure %}
user> (kill-server)
:killed
user> (boot-server)
{% endhighlight %}

Alternatively, you can connect to an external server - this is more
portable and robust, since the internal server can't run everywhere
and will bring down the whole JVM if it crashes. On the other hand,
external servers are more tricky to connect and have slower access to
buffers. I always use the internal server, but [see
here](https://github.com/overtone/overtone/wiki/Connecting-scsynth#connecting-to-an-external-server)
for more information about using external servers.

UGens
=====

Unit generators (UGens) are the one of the major building blocks in
Overtone/Supercollider.  They can have multiple inputs and outputs,
and can also perform side effects like writing to files.  A vast array
of them are distributed with Overtone, from oscillators to spectrum
analyzers - there's a handy complete listing on pages 3 and 4 of the
[overtone cheat sheet](https://github.com/overtone/overtone/raw/master/docs/cheatsheet/overtone-cheat-sheet.pdf).

Many of the inputs/outputs to UGens represent audio data, some deal
with other kinds of non-audio data such as control signals. UGens can
run at either audio rate, or control rate (~1/60th audio
rate). Although you could run everything at audio rate, running
control UGens at control rate saves CPU. By default, Overtone will
attempt to automatically choose the correct rate for your UGens, but
you can manually specify them using the suffixes `:ar` and `:kr` as
follows - use these liberally as they will avoid headaches when Overtone
inevitably guesses the wrong one.

{% highlight clojure %}
user> (demo (impulse))  ; no sound because impulse defaults to :kr, which is inaudible
user> (demo (impulse:kr)) ; same as above
user> (demo (impulse:ar)) ; now we hear 440Hz
{% endhighlight %}

`demo` is a handy function that creates an anonymous synth and plays
it for a few seconds. We'll look at synths in the next section.

Another example is the sine wave UGen `sin-osc`. it takes frequency
and phase arguments. Most UGens have default arguments (given in their
doc string) and you can specify named arguments using keywords. `sin-osc`
defaults to 440Hz and 0 phase, so these are all equivalent:

{% highlight clojure %}
user> (demo (sin-osc))  ; use default args, 440Hz, 0 phase
user> (demo (sin-osc 440 0)) ; same as above
user> (demo (sin-osc :phase 0))
user> (demo (sin-osc :phase 0 :freq 440)) ; keywords let us specify args in any order
{% endhighlight %}

So anyway, what are those "synths" I mentioned?

Synths
======

When producing sound, the server does not deal with UGens directly but
rather synths.  Synths are combinations of one or more UGens, and are
represented internally as graphs of UGens.

You can create a synth using the `synth` macro. As a lame example,
let's consider a synth that produces a sine wave of a random frequency
(between 200 and 2000Hz) that changes 10 times per second. The
`t-rand` UGen is generates a random float when it's trigger input goes
positive, and the `impulse` UGen goes positive once per cycle, so
putting it together...

{% highlight clojure %}
user> (synth [rate 10] (sin-osc:ar (t-rand:kr 200 2000 (impulse:kr rate))))
#overtone.sc.synth.Synth{:name "anon-6", :ugens nil, :sdef {:name "user.core/anon-6", :constants [0.0 10.0 200.0 2000.0], :params (), :pnames (), :ugens (#<sc-ugen: impulse:kr [0]> #<sc-ugen: t-rand:kr [1]> #<sc-ugen: sin-osc:ar [2]>)}, :args (), :params (), :instance-fn #<core$identity clojure.core$identity@7b77f35e>}
{% endhighlight %}

Note that we can optionally specify a binding for the synth body, here
we bind 10 to rate.  Hmm, no sound. We forgot to include the `out`
UGen which writes the data to our sound hardware!  Also, we actually
need to call the record returned by `synth` in order to trigger the
synth, so we need some extra parentheses.

{% highlight clojure %}
user> ((synth (out 0 (sin-osc:ar (t-rand:kr 200 2000 (impulse:kr 10)))))
#<synth-node[loading]: music.core/anon-10 271>
user> (stop)
{% endhighlight %}

Success. The 0 argument to `out` is the number of the output bus to
use. The lowest numbers go to your hardware, but you can create your
own buses for routing internally - I'll cover this in another post.
To stop the output, we just call `stop` - this stops all running synths.

For testing, `demo` is more convenient because it automatically wraps
UGens in an `out` and `synth`. In practice, there is little need to
use the `synth` macro at all since overtone provides the more useful
`defsynth` and `definst` macros for creating reusable named synths.

{% highlight clojure %}
user> (defsynth rand-sin [rate 10] (out 0 (sin-osc:ar (t-rand:kr 200 2000 (impulse:kr rate)))))
#<synth: rand-sin>
user> (rand-sin) ; it works!
#<synth-node[loading]: user.core/rand-sin 279>
user> (stop)
user> (rand-sin 20) ; we can even change the rate
#<synth-node[loading]: user.core/rand-sin 280>
user> (stop)
{% endhighlight %}

`definst` is even handier, because it automatically wraps your synth
in an `out`. They have some limitations such as restriction to 1 or 2
channels but also have nice features like the ability to easily chain
effects onto them - see the doc string for full details.

{% highlight clojure %}
user> (definst rand-sin-inst [rate 10] (sin-osc:ar (t-rand:kr 200 2000 (impulse:kr rate))))
#<instrument rand-sin-inst>
user> (rand-sin-inst) ; it works, no out required!
{% endhighlight %}

Finally, you can visualize the UGen graph of a synth using
`show-graphviz-synth` - this is useful for documenting synths.
Here's what rand-sin looks like.

{% highlight clojure %}
user> (show-graphviz-synth rand-sin)
{% endhighlight %}

![rand-sin graph](https://raw.githubusercontent.com/mattearnshaw/mattearnshaw.github.io/master/assets/rand-sin-graph.png)

In the next post I'll probably cover buses and buffers.

[overtone]: http://overtone.github.io/
[clojure]: http://clojure.org/
[supercollider]: http://supercollider.sourceforge.net/
[at-at]: https://github.com/overtone/at-at
[examples]: https://github.com/overtone/overtone/tree/master/src/overtone/examples
[CIDER]: https://github.com/clojure-emacs/cider
