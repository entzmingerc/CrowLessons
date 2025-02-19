# --- Goal of the Lesson ---

Simple synth voice. We want a v/oct sequence, an AR envelope, and an ADSR envelope.  

Crow Input 1 set to 'change' input mode. When voltage at Crow input 1 rises past 1 V, it will step the `volt_seq` of notes, trigger an AR envelope on output 2, and trigger an ADSR envelope on output 3. When voltage at Crow input 1 falls past 1 V, it will trigger the release stage of the ADSR on output 3. 

```Lua
s = sequins
volt_seq = s{1,2,3,4}/12

function step_sequence(v)
    if v == true then
        val = volt_seq()
        output[1].volts = val 
        output[2]()
        output[3](v)
    else
        output[3](v)
    end
end

function init()
    input[1].mode('change', 1, 0.1, 'both')
    input[1].change = step_sequence
    output[2].action = ar()
    output[3].action = adsr()
end
```

# --- Make sure druid can upload code to crow ---

druid should be installed in the same folder as your Lua code. Let's make a new .lua file with a function named `init()`. I believe if `init()` is present in a file, then it will be the first function to be called after global variables and commands have been executed. 
```Lua
function init()
	print("CAW!")
end
```

Save your .lua file as lesson01.lua and then run druid. Connect to Crow from your computer using a USB cable. Type `u lesson01.lua` to upload the lua file to Crow. If successful you should see the following output.
```
> u lesson01.lua            
uploading lesson01.lua 
User script updated.
Running: function init()
CAW!
^^ready()
```
We can now make edits to lesson01 and upload the edited file to Crow.  


# --- Simple event / response using input modes ---

### Input Mode Change

Let's set up an input using the input mode 'change'.  

Crow can address `input[1]` or `input[2]` and change the behavior of how Crow processes voltage at the inputs. There are a bunch of different behaviors Crow can do with the input voltage. Let's look at 'change' to start.  
https://monome.org/docs/crow/reference/#input-modes  

`input[n].mode( 'change', threshold, hysteresis, direction )`  

Let's discuss what the `change` mode does. The first argument is the name of the mode, in this case we're using 'change' to call a function whenever the voltage at the input passes by a certain value. If we set `threshold` to 1, then each time we pass by 1 V then we call a function. The argument `direction` can be set to rising, falling, or both. For rising, if we rise in voltage from 0 to 2 V going upwards, then it will call a function. For falling, if we fall in voltage from 2 to 0 V going downwards, then it will call a function. For both, then a function will be called for both rising and falling cases. If we have some noise on our input voltage, then we might want to have a small `hysteresis` range to make sure we only trigger the function once. If we set hysteresis to 0.1 V for example, then it will only call a function when we pass from 1.95 V to 2.05 V.  

Many (if not all) of the Crow functions have defaults. If curious, you can look them up in the code directly on GitHub. https://github.com/monome/crow/blob/main/lua/input.lua For now, let's just type this into our init() function.  

`input[1].mode( 'change', 1, 0.1, 'rising')`  

This will call a function whenever our voltage rises past 1 V, but what function are we calling? I think the proper terminology is that we are triggering an "event" and we need a function to "handle" the event. In the Crow Reference, https://monome.org/docs/crow/reference/#event-handlers--callbacks there is a section that describes these event handlers. Let's add the following line to our init() function.  

`input[1].change = print_caw`  

The above line of code says that whenever the `change` event occurs on input 1, we will call the function "print_caw". Let's write print_caw now  

```Lua
function print_caw()
	print("CAW!")
end
```

Great. Now all together our entire lesson01.lua file should be  

```Lua
function print_caw()
    print("CAW!")
end

function init()
    input[1].mode('change', 1, 0.1, 'rising')
    input[1].change = print_caw
end
```

Save your .lua file, then use druid to upload this file (`u lesson01.lua`) to crow. Now patch a slow LFO to Crow Input 1. The output on druid should be printing CAW! whenever the voltage rises past 1 V.  

The input mode change is great for gate inputs. If you have a square wave LFO, a trigger source, or a gate source from a midi controller and you want to have an function occur whenever a voltage goes high, then you can use 'change' to respond immediately to this change in voltage.  
### Input Mode Stream

Now, let's use input mode `stream` instead of change. Stream will call a function on a timer. For example, it could call print_caw every 2 seconds. Look at how to use stream in the Crow Reference.  
https://monome.org/docs/crow/reference/#input-modes  

`input[n].mode( 'stream', time )`  

Stream only has one argument compared to change and that is the time. We can set the function that stream calls in a similar way to how we did it with change. Edit your init() function to use stream instead of change.  

```Lua
function print_caw()
    print("CAW!")
end

function init()
	input[1].mode( 'stream', 2)
	input[1].stream = print_caw
end
```

Save your .lua file, then use druid to upload this file (`u lesson01.lua`) to crow. Every 2 seconds, Crow should print CAW! to the screen on druid. Unplug your LFO that is patched into input 1. Note how we don't need an input voltage to call a function if we're using stream. The nice part about stream though is that it can read the voltage at input 1 and use it inside of the function that it is calling.  

To demonstrate how stream can use the input voltage, let's print it every 2 seconds. Look at this line of code in the init() function:  

`input[1].stream = print_caw`  

This is setting the variable `input[1].stream` to the value of `print_caw`. In Lua, a notable thing about the language is that it can treat function as variables. Instead of setting this to the value of print_caw, we could write a function directly like so:  

```Lua
function print_caw()
    print("CAW!")
end

function init()
	input[1].mode( 'stream', 2)
	input[1].stream = function(v)
		print(v)
	end
end
```

Now we have made a new function that just prints the variable `v` and we are calling it every 2 seconds. This is a special function of Crow specifically when using stream. The value of variable `v` is the voltage read at Crow input 1 whenever the function is called. If we save and upload this code to Crow now, we should see values close to zero being printed to the screen if you do not have any voltage signal patched to input 1 on Crow. If you patch a slow LFO to Crow input 1, you should see the numbers going up and down in value.  

Using input mode stream is a good way to call a function routinely based on the input voltage. For example, you could set the time to 0.01 seconds and read the input voltage very frequently. You could read in a voltage of 1 V and  output a voltage twice as large, or you could read input 1 voltage and do something if you ever saw the input voltage drop below -2 V, or anything else you could think. The key point is that you'd be calling this function every 0.01 seconds, which may eat up a lot of computation. I think the fastest you can go with stream is 0.0015 seconds.  

You could approximate the behavior of 'change' using 'stream', however you'd have to run stream very quickly and waste a lot of CPU when change would probably respond faster to changes in voltage anyway. This demonstrates some of the different use cases between each of the input modes. `Window` calls a function whenever a voltage at the input enters a specified range of voltages. `Scale` is similar to `Window` but it calls an event whenever the input voltage enters a new voltage range for a note. `Volume` is similar to `Stream` but instead of reading the input voltage at a point in time, it tracks the RMS amplitude of an audio rate signal, which is useful if you're trying to make an envelope follower or have something react to the volume of your audio.  

# --- Simple Synth Voice ---
For a simple synth voice, we want to output 2 CV signals: a v/oct signal and an envelope (AR or ADSR). For a v/oct signal, we'll need a melody to play as well, which is where Sequins can be useful. For the AR or ADSR envelope we'll need to use Crow's ASL language. For both of these goals, we'll need to know how to use the outputs on Crow. We can use the input to trigger the envelopes and step the melody sequence.  

### Use Input Mode Change to Step Through a Voltage Sequence

Let's start with our previous example using input mode 'change'.  

```Lua
function print_caw()
    print("CAW!")
end

function init()
    input[1].mode('change', 1, 0.1, 'rising')
    input[1].change = print_caw
end
```

If we patch a gate or square wave LFO to input 1, then it will print CAW! each time the voltage goes high. Instead of printing a message, we want to step through a sequence of voltages. We want to manually iterate through a sequence of voltage values and set output 1 to each voltage value. We will first create a simple step sequence manually, then we can use Sequins as a replacement.  

Let's delete print_caw() and write a new function called step_sequence():  

```Lua
idx = 0
volt_seq = {1, 2, 3, 4}

function step_sequence()
	idx = idx + 1
	if idx > 4 then 
		idx = 1 
	end
    output[1].volts = volt_seq[idx]
    print(volt_seq[idx])
end

function init()
    input[1].mode('change', 1, 0.1, 'rising')
    input[1].change = step_sequence
end
```

Note that we have set the `input[1].change` to our new function step_sequence. We have two new global variables: `idx` which is short for "index" and `volt_seq` which is a table representing our sequence of voltage values. When we call step_sequence, we first add 1 to our index. If our index is above 4, then we need to reset back to 1. Finally, we look up the value of our voltage sequence at the index and set output 1 volts to the value we find there. When writing code I also like to print values to make sure I see the code working. If we save and upload this to Crow we should see 1, 2, 3, 4, 1, 2, 3, 4, 1 ... being printed on druid and the output voltage should be jumping between 1 V, 2 V, 3 V, and 4 V.  

Output voltages by default can have their `volts`,  `slew`, `shape` parameters set. For now, we are just setting the `volts` of output 1 to a value. This is the default ASL behavior of each of the outputs. If you change the ASL on an output, then you may not be able to set these parameters any more, but you can define what parameters you can set using ASL.  
https://monome.org/docs/crow/reference/#setting-cv  

Now we are going to replace our sequence with sequins. This allows us to replace both the voltage sequence and our index with just a sequins table.  
https://monome.org/docs/crow/reference/#sequins  

First we can alias the sequins object to the variable s which makes it easier to type. This is just a style preference that Trent (the creator of Crow) likes to use.  

`s = sequins`   

Next we can use `s` to recreate our 4 step sequence.  

`volt_seq = s{1, 2, 3, 4}`  

Now we have a sequins called `volt_seq`. Whenever we call volt_seq() like a function, it will return the next value of the sequence. Check out the following example:  

```Lua
s = sequins
volt_seq = s{1, 2, 3, 4}
print(volt_seq()) -- this prints: 1
print(volt_seq()) -- this prints: 2
print(volt_seq()) -- this prints: 3
print(volt_seq()) -- this prints: 4
print(volt_seq()) -- this prints: 1
```

We can replace idx and volt_seq with a sequins as follows:  

```Lua
s = sequins
volt_seq = s{1,2,3,4}

function step_sequence()
    val = volt_seq()
    output[1].volts = val
    print(val)
end

function init()
    input[1].mode('change', 1, 0.1, 'rising')
    input[1].change = step_sequence
end
```

Save and upload this code and you should see no change in behavior as before, but now we are using sequins to step through the 4 voltage values. Now we don't have to worry about checking the index or indexing the table, all we do is call volt_seq() and get a number back! There's a number of different things we can do to modify the sequins:  

```Lua
volt_seq = s{1,2,3,4,5}
-- 1, 2, 3, 4, 5, 1, 2, 3, ...

volt_seq[1] = 8 -- you can change the values like a table
-- 8, 2, 3, 4, 5, 8, 2, 3, ...
print(volt_seq[1]) -- 8

volt_seq = s{1,2,3,4,5}:step(2) -- step by 2 values instead of 1
volt_seq:step(2) -- this does the same thing
-- 1, 3, 5, 2, 4, 1, 3, 5, ...

volt_seq = s{1, 2, s{3, 30}, 4, 5} -- you can nest sequins!
-- 1, 2, 3, 4, 5, 1, 2, 30, ...
```

With a v/oct signal, we know that for every 1 volt increase, that corresponds to one octave increase. If we output 1 V, 2 V, 3 V, and 4 V this is stepping through 4 different octaves. Instead, we want to step through notes. "12TET" divides each octave into 12 different notes. We can do this with sequins with sequins transformers.  
https://monome.org/docs/crow/reference/#sequins-transformers  

```Lua
volt_seq = s{1,2,3,4,5}
output[1].volts = volt_seq() / 12 -- each value is divided by 12 to make a note

-- instead of dividing by 12 at the output, we could divide the sequins
volt_seq = s{1,2,3,4,5}/12 -- this divides each value by 12
volt_seq = s{1/12, 2/12, 3/12, 4/12, 5/12} -- same as previous line!
volt_seq = s{1,2,3,4,5}:map(function(n) return n/12 end) -- same thing!

print(seq) -- prints the sequins
#volt_seq -- returns length of sequins, each nested sequins counts for 1 element
volt_seq:peek() -- returns the current value, without advancing the sequins
volt_seq = s{1,2,3,4,5} + s{13,14,15,16,17} -- results in a 10 element sequins
-- 1,2,3,4,5,13,14,15,16,17,1,2,3,4,...
```

Modify your volt_seq as follows:  

```Lua
volt_seq = s{1,2,3,4}/12
```

Then patch Crow output 1 to the v/oct input of an oscillator. Patch the output of your oscillator to your speakers or headphones some how and now whenever a gate input goes high on Crow input 1, you'll hear your step sequence iterate through the sequins and change the note being played by your oscillator.   

A cool thing about Crow is that you can change global variables on the fly. In druid, try typing the following:  

`volt_seq = s{4, 1, 6, 3, 2}/12`  

This should overwrite the existing sequence and you can hear the new sequence playing.   

### Trigger an AR envelope when input 1 goes high

Whenever the voltage goes high on input 1, let's trigger an AR and ADSR envelope. The AR envelope will move through the attack and release stages then hold at the ending value. The ADSR envelope will move through the attack and decay stages and hold at the value of the sustain stage until the input on 1 goes low.  

First we'll add an AR envelope to output 2.  
What is ASL? https://monome.org/docs/crow/reference/#asl  
What are some built-in ASL tables? https://github.com/monome/crow/blob/main/lua/asllib.lua  

ASL is an acronym for "a slope language". You can think of this as "defining the output behavior". Instead of setting output voltages statically, you can set outputs to have dynamic, time-varying voltages like LFOs, envelopes, and many more. There are 6 built in ASL shapes: lfo, oscillate, pulse, ramp, ar, and adsr (see second link above). These 6 example ASL shapes I have referenced a million times to figure out how to write ASL shapes, so I think it's a pretty important link. There is no way to manipulate the output voltages of Crow except by setting and manipulating the ASL on an output.  

We have a built in AR ASL shape. We can set the ASL on an output as follows  
https://monome.org/docs/crow/reference/#action  

```Lua
output[2].action = ar() -- sets the action of an output
output[2]() -- trigger the action of an output
```

Let's look at AR in the asllib.lua  
https://github.com/monome/crow/blob/4ca8b7d1b6ad8d07b368bcf18244054a5de0747c/lua/asllib.lua#L48  

```Lua
function ar(a, r, level, shape)
    a = a or 0.05
    r = r or 0.5
    level = level or 7
    shape = shape or 'log'

    return{ to(level, a, shape)
          , to(0, r, shape)
          }
end
```

Okay let's go line by line. First there are the arguments. We have 'a' which is attack time, r is release time, level is amplitude of the envelope, and shape is how we get from one voltage to the next. If you call `ar()` without any values then all the arguments will default to `nil`. The first line `a = a or 0.05` will return `a` if it's not nil, but if it is nil, then it will return 0.05 which is the default value. The same logic applies to the next 3 lines.  

The return statement is where ASL is constructed. ASL is a table of function calls. ASL is usually a sequence of `to()` function calls. The first argument of to() is the voltage that we're going to, the destination voltage. The second argument of to() is the time it takes to get to the voltage (in seconds). The third argument is the path we take to get to that voltage.  

For example, let us assume we start at 0 V. We want to rise to 5 V over 1 second, then we want to fall to 0 V over 1 second. We want to have a straight line between each voltage, so let's use a 'linear' CV shape. We could write this ASL shape as follows:  

```Lua
{to(5, 1, 'linear'), to(0, 1, 'linear')}
```

If we return to our ar() function above, let's rewrite the return statement with the default values of the ar() function:  

```Lua
return {to(7, 0.05, 'log'), to(0, 0.5, 'log')}
```

So this is the ASL we're passing to output 2's action. Let's add this to our running code for lesson01 as follows:  

```Lua
s = sequins
volt_seq = s{1,2,3,4}/12

function step_sequence()
    val = volt_seq() -- get next value in sequence
    output[1].volts = val -- set output voltage
    output[2]() -- trigger ar()
    print(val)
end

function init()
    input[1].mode('change', 1, 0.1, 'rising') -- event when passing 1 V
    input[1].change = step_sequence -- when triggered, call this function
    output[2].action = ar() -- set output 2 action to ar()
end
```

In our init() function, we have set the action of output 2. In our step_sequence() function, we trigger output 2's action whenever the gate on input 1 goes high. Patch output 2 to something that likes AR modulation, such as the VCA to create an amplitude envelope for your oscillator, or to filter cutoff for modulation.  

### Trigger ADSR envelope when input 1 goes high and release when input 1 goes low

We're almost done, let's do exactly the same thing but for output 3 using the built-in ADSR function. So far, we have only been doing things when input 1 goes high, only during the "rising" stage. Now we need to trigger the release stage of the ADSR envelope whenever the input 1 goes low as well. We need to modify the input mode 'change' to call step_sequence when it goes high and when it goes low by replacing 'rising' with the 'both' option.  

```Lua
input[1].mode('change', 1, 0.1, 'both')
```

Now when our square wave at input 1 passes from above 1 V to below 1 V, we'll trigger step_sequence. Do you remember how we could use the stream input mode and pass the input voltage `v` into the function that we were calling so we could print it? We can do something similar with change! If we modify step_sequence to have an argument `v` then it will be true or false depending on if we are rising (true) or falling (false). Crow will pass the value of true or false to the first argument of the function. We can modify our code as follows:  

```Lua
s = sequins
volt_seq = s{1,2,3,4}/12

function step_sequence(v)
    val = volt_seq() -- get next value in sequence
    output[1].volts = val -- set output voltage
    output[2]() -- trigger ar()
    print(v, val) -- print both the stage and the sequins value
end

function init()
    input[1].mode('change', 1, 0.1, 'both') -- trigger when rising or falling
    input[1].change = step_sequence -- if triggered, call this function
    output[2].action = ar() -- set output 2 action to ar()
end
```

Save and load this code and note how we have true and false values being printed to the screen on druid along with the value of our voltage sequence.  

If v is true, we just do what we were already doing. If v is false, that means we're in the release stage for the ADSR. We need to set output 3's action to adsr() similar to how we set output 2's action to ar(). How do we "gate" the ADSR function? What does that mean in terms of ASL?  

Check out the ADSR function in asllib.lua  
https://github.com/monome/crow/blob/4ca8b7d1b6ad8d07b368bcf18244054a5de0747c/lua/asllib.lua#L59C1-L70C4  

```Lua
function adsr(a, d, s, r, shape)
    a = a or 0.05
    d = d or 0.3
    s = s or 2
    r = r or 2

    return{ held{ to(7.0, a, shape)
                , to(s, d, shape)
                }
          , to(0, r, shape)
          }
end
```

This is similar to the ar() function, but here we have a `held{}` thing inside the ASL. In the Crow Reference we see held mentioned in the ASL section  
https://monome.org/docs/crow/reference/#asl  
```Lua
held{ <asl> } -- freeze at end of sequence until a false or 'release' directive
```
This means that the voltage in the ASL sequence will stop at the end of the second to() stage until we give the ASL a `false` value. In the Crow Reference there is also a simple example of using the ADSR shape here:  
https://monome.org/docs/crow/reference/#action  
```Lua
output[1].action = adsr()
output[1]( true )  -- start attack phase and pause at sustain
output[1]( true )  -- re-start attack phase from the current location
output[1]( false ) -- enter release phase
```
This will trigger the attack stage, move through the decay stage (which is the second to() function), then hold at sustain. If we send `true` again, it will reset from attack stage. If we send `false`, then we'll move to the release stage. Let's rewrite the ADSR ASL using the default values:  

```Lua
{ held{to(7.0, 0.05, 'linear'), to(2, 0.3, 'linear')}, to(0, 2, 'linear')}
```

The attack stage goes up to 7 V over 0.05 seconds, the decay stage falls from 7 V to 2 V over 0.3 seconds, then held{} makes the ASL sustain at 2 V until `false` is received where it then falls to 0 V over 2 seconds in the release stage.  

Let's rewrite our code to use the adsr() function on output 3 and use the true and false data from the input mode to trigger the release stage whenever the gate on Crow input 1 goes low.  

```Lua
s = sequins
volt_seq = s{1,2,3,4}/12

function step_sequence(v)
    if v == true then
        val = volt_seq() -- get next value in sequence
        output[1].volts = val -- set output voltage
        output[2]() -- trigger ar()
        output[3](v) -- v is true if we're here
        print(v, val)
    else
        output[3](v) -- v is false if we're here
        print(v)
    end
end

function init()
    input[1].mode('change', 1, 0.1, 'both') -- trigger when rising or falling
    input[1].change = step_sequence -- if triggered, call this function
    output[2].action = ar() -- set output 2 action to ar()
    output[3].action = adsr() -- set output 3 action to adsr()
end
```

In our init() function we set the action of output 3 to the default adsr envelope and we modify the 'change' function to use the 'both' option so that we trigger step_sequence when we rise past 1 V or when we fall past 1 V. In step_sequence function, if v is true then we set output 1 to our next voltage value, trigger the AR envelope on output 2, and trigger the A, D, and S stages of our ADSR envelope on output 3. Then, when our gate goes low and triggers step_sequence again, v will be false and we'll pass that false value to output 3 which should trigger the release stage of our ADSR envelope.  

We've reached the goal for the lesson!  

From here we could edit the attack and release times of the ADSR envelope by passing in different values when we initialize output 3's action in the init() function. Or we could modify the attack and release times of ar(). Or we could create different sequence of voltages using sequins. Or we could replace ar() with lfo() and explore using LFOs and the loop{} command in ASL. If you're curious, just follow the same steps we used to explore how to use ar() by looking up lfo in asllib.lua and adjusting values you pass to it. Or you could write your own ASL from scratch, maybe you could add 3 or 4 more to() stages to the ADSR ASL?  

Thanks for reading! CAW!  

