Download Link: https://assignmentchef.com/product/solved-comp2300-lab-10
<br>
Before you attend this week’s lab, make sure:

<ol>

 <li>you have completed <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/09-input-through-interrupts/">lab 9</a> and understand how signals (i.e. voltage changes) on your GPIO pins can trigger interrupts</li>

</ol>

In this week’s lab you will:

<ol>

 <li>configure the GPIO pins on your board for both input and output</li>

 <li>connect the GPIO pins to one another with physical wires</li>

 <li>configure and write interrupt handlers to <em>do things</em> when stuff happens on these wires</li>

 <li><em>simulate</em> a multiboard setup where you connect the two sides of your discoboard to each other with wires and turn the LEDs on from either side</li>

</ol>

<h2 id="introduction">Introduction</h2>

This week you’ll take a deeper dive into the GPIO &amp; interrupt capability on your discoboard. The <strong>GP</strong> in <strong>GP</strong>IO stands for <strong>G</strong>eneral <strong>P</strong>urpose, which means that each pin (the pointy little gold-coloured bits of metal sticking up in rows along the sides of your discoboard) can be used for either digital input <em>or</em> output (or even other things). The mode (input mode, output mode, alternate mode) of a given pin is configured (you guessed it!) by writing certain bits to special GPIO configuration registers.

In this week’s lab you’ll learn more about the stuff you did in Week 8 when you set the joystick pins to input mode, and you’ll learn how to control the signals (i.e. high and low voltages) coming out of the pins in software.

<a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/10-connecting-discoboards/#exercise-1">Exercise 1</a> lays the groundwork (and is pretty wordy) but then things pick up in <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/10-connecting-discoboards/#input-with-interrupts">Exercise 2</a>, and by the end of the lab you’ll get to be really creative. This lab might seem a bit similar to <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/09-input-through-interrupts/">last week</a>, but it goes into a bit more depth about exactly what it means for you to configure, read &amp; write to the GPIO pins, which is going to be super-helpful for assignment 3.

<p class="talk-box">Today you’ll be working on sending signals to and from GPIO pins by connecting these pins using jumper cables. To start off, think about: in the context of GPIO pins, what is a <em>signal</em>? Is there a difference between a signal which comes internally through the discoboard (e.g. the joystick input you used in the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/09-input-through-interrupts/">Week 9 lab</a> and the signal which comes in through an external wire?

<h2 id="exercise-1">Exercise 1: click-to-blink recap</h2>

This first exercise will <em>seem</em> like a re-hash of <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/06-blinky/">stuff you’ve</a> <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/09-input-through-interrupts/">done before</a>—clicking the joystick and turning on an LED.

However, this time when you fork &amp; clone the <a class="acton-tabs-link-processed" href="https://gitlab.cecs.anu.edu.au/comp2300/2021/comp2300-2021-lab-10">lab 10 template</a> and have a look inside there’s some good news and some bad news:

<ul>

 <li>the <strong>bad news</strong> is that you don’t have the sweet <code>joystick.S</code> and <code>led.S</code> helper libraries from last time<sup id="fnref:files" role="doc-noteref"><a class="footnote acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/10-connecting-discoboards/#fn:files">1</a></sup></li>

 <li>the <strong>good news</strong> is that you’ll get to write it yourself and see exactly how it works (and there are still some helpful utility functions &amp; macros in the template—you don’t have to start <em>completely</em> from scratch)</li>

</ul>

As you re-do the LED &amp; joystick configuration using a generic GPIO config process you’ll get a better picture of how the GPIO infrastructure and NVIC/EXTI interrupt controllers work together on your discoboard.

<h3 id="gpio-output">GPIO <strong>output</strong></h3>

To configure a pin in output mode, <code>src/libcomp2300/macros.S</code> has a handy <code>GPIO_configure_output_pin</code> macro. For example, to declare pin <code>PH0</code> as an output pin, you could call the macro like so:

<pre><code class="language-ARM hljs"><span class="hljs-symbol">GPIO_configure_output_pin</span> H, <span class="hljs-number">0</span></code></pre>

<p class="info-box">Most of the macros in the <code>macros.S</code> are just simple wrappers around function calls, using the macro to set up correct parameters (remember, <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/lectures/week-6-macros/#what-are-macros-for">macros are just kids with scissors!</a>). You still need to be aware (and careful) of which registers the macros might touch. Before you use any macros in this lab, <strong>make sure you read the macro definition and understand what it is doing</strong>.

To use the GPIO pins, you need to turn them on by making sure the corresponding GPIO port receives a clock signal. Remember from <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/06-blinky/">lab 5</a> that the discoboard’s red and green LEDs are connected to ports B and E respectively:

<pre><code class="language-ARM hljs"><span class="hljs-symbol">GPIOx_clock_enable</span> <span class="hljs-keyword">B</span><span class="hljs-symbol">GPIOx_clock_enable</span> E</code></pre>

Make sure you enable port <code>A</code> as well for joystick.

To send data out (i.e. to change the voltage) on the GPIO pin, you write a <code>0</code> or <code>1</code> to the GPIO port’s Output Data Register (ODR). Remember that the red LED is connected on GPIO <code>PB2</code> and the green LED is on <code>PE8</code>. So you can turn on the LEDs by writing a <code>1</code> to their output data registers. Most of these helper macros have <code>set</code>, <code>clear</code> and <code>toggle</code> versions, which do what you’d expect.

<pre><code class="language-ARM hljs"><span class="hljs-symbol">GPIO_configure_output_pin</span> <span class="hljs-keyword">B, </span><span class="hljs-number">2</span> <span class="hljs-comment">@ (red LED)</span><span class="hljs-symbol">GPIO_configure_output_pin</span> E, <span class="hljs-number">8</span> <span class="hljs-comment">@ (green LED)</span><span class="hljs-symbol">GPIOx_ODR_set</span> <span class="hljs-keyword">B, </span><span class="hljs-number">2</span><span class="hljs-symbol">GPIOx_ODR_set</span> E, <span class="hljs-number">8</span></code></pre>

<p class="think-box">Once you write a signal (a <code>0</code> or <code>1</code>) to the GPIO line, how long does it “stay there” for? How could you figure this out?

<h3 id="gpio-input">GPIO <strong>input</strong></h3>

All good so far—remember that this is just a recap. Now, let’s take the same approach to the joystick, which is similar to the LEDs in that it’s wired to specific GPIO pins (<code>PA0</code>: centre, <code>PA1</code>: left, <code>PA5</code>: down, <code>PA2</code>: right, <code>PA3</code>: up) but this time it’s an <strong>input</strong> device. There are some macros for this, too:

<pre><code class="language-ARM hljs"><span class="hljs-symbol">GPIO_configure_input_pin</span> A, <span class="hljs-number">0</span> <span class="hljs-comment">@ (central joystick button)</span></code></pre>

Here, you’ve declared the pin <code>PA0</code> as an input pin. The <code>GPIO_configure_input_pin</code> macro does a couple of things:

<ul>

 <li>sets the <em>input</em> bit pattern into the appropriate mode register (it’s <code>0b00</code> for input, just like it was <code>0b01</code> for output)</li>

 <li>configures the input pin to use a <a class="acton-tabs-link-processed" href="https://learn.sparkfun.com/tutorials/pull-up-resistors/what-is-a-pull-up-resistor">pull-down resistor</a>—this is so that it will reliably read a <code>0</code> even if it’s not connected to anything (if you don’t do this, then you might get weird results when your input isn’t connected to anything)</li>

</ul>

To read data in from a GPIO pin, you can read the current value (high <code>1</code> or low <code>0</code>) on any GPIO pin at any time by reading the appropriate bit from the GPIO port’s Input Data Register (IDR). There’s a helper macro for this as well—<code>GPIOx_IDR_read</code>, and it sets flags based on the result (so the <strong>z</strong>ero flag will be set if the GPIO line is low, and it won’t be set if the GPIO line is high).

You can do this as often as you like—reading data from the pin with <code>GPIOx_IDR_read</code> will always leave the current value (<code>0</code> or <code>1</code>) in <code>r0</code> and also set the flags appropriately, and it doesn’t change the signal on the pin. You can use this to “poll” a given pin in a loop:

<pre><code class="language-ARM hljs"><span class="hljs-symbol">poll_gpio</span>:  <span class="hljs-comment">@ read PA0, set flags based on result</span>  GPIOx_IDR_read A, <span class="hljs-number">0</span>    <span class="hljs-comment">@ do something based on the flags in here</span>    <span class="hljs-keyword">b </span>poll_gpio</code></pre>

<p class="push-box">Write a program which enables the central joystick button as an input, and polls (as in the loop above) to turn the green LED on when the button is pressed. Commit &amp; push your program to GitLab.

<h2 id="input-with-interrupts">Exercise 2: let’s do it again, this time using interrupts</h2>

As we talked about in the week 8 lectures, polling the current value on the pin in a loop isn’t the best way to do things, because it makes it hard to do other stuff in the meantime. There’s a better way: configure the GPIO line to fire an interrupt when the value changes.

Before you can enable and configure the interrupts, you need to enable the System Configuration Controller (<code>SYSCFG</code>) clock so that you can modify the system configuration (see Chaper 8 and 8.2.3 in the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/v_media/manuals/stm32-L476G-discovery-reference-manual.pdf">discoboard reference manual</a>).

<pre><code class="language-ARM hljs"><span class="hljs-comment">@ enable SYSCFG clock</span><span class="hljs-symbol">RCC_APB2ENR_set</span> <span class="hljs-number">0</span></code></pre>

This code is included in the <a class="acton-tabs-link-processed" href="https://gitlab.cecs.anu.edu.au/comp2300/2021/comp2300-2021-lab-10">template</a>, you can see the relevant configuration register in Section 6.4.21 on p233 of the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/v_media/manuals/stm32-L476G-discovery-reference-manual.pdf">discoboard reference manual</a>.

There’s one other macro in <code>macros.S</code> which is helpful when setting up GPIO pins as input: <code>GPIO_configure_input_pin_it</code>—note the <code>_it</code> suffix. This macro does all the configuration of <code>GPIO_configure_input_pin</code> and additionally registers the pin as a source for interrupts. There are two parts of the discoboard which are working together to do this:

<ul>

 <li>the <em>Extended Interrupts and Events Controller</em> (EXTI) is the part of your discoboard which allows signals (either a rising or falling edge) on the GPIO pins to trigger an interrupt</li>

 <li>the <em>Nested Vectored Interrupt Controller</em><sup id="fnref:nvic" role="doc-noteref"><a class="footnote acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/10-connecting-discoboards/#fn:nvic">2</a></sup> (NVIC) is the hardware which “receives” the interrupt, and (depending on the priority, what other interrupts are running, and a few other things) will interrupt the CPU and transfer control to the appropriate handler function in the vector table</li>

</ul>

The <code>GPIO_configure_input_pin_it</code> sets the appropriate bits to enable the the pin as a source of interrupts. The next step in the EXTI configuration is to determine whether the interrupt will be triggered on a <strong>rising edge</strong> (<code>0</code> to <code>1</code> transition) or a <strong>falling edge</strong> (<code>1</code> to <code>0</code> transition). Again, there are helper macros for this:

<pre><code class="language-ARM hljs"><span class="hljs-symbol">GPIO_configure_input_pin_it</span> A, <span class="hljs-number">0</span><span class="hljs-symbol">EXTI_set_rising_edge_trigger</span> <span class="hljs-number">0</span><span class="hljs-symbol">EXTI_set_falling_edge_trigger</span> <span class="hljs-number">0</span></code></pre>

There are a couple of quirks here: firstly, the EXTI controller can only listen to one port for a given pin number, so for example you can’t have both <code>PA0</code> and <code>PB0</code> triggering an interrupt. This is because the pins are multiplexed in the EXTI controller (see the diagram below). This is to keep things simple—you probably don’t <em>need</em> separate interrupt triggers on all the pins. Here’s an example of what this looks like for EXTI0—pin zero from <em>all</em> ports goes in there, and the EXTI controller can only listen to one at a time.

<img decoding="async" alt="GPIO to EXTI mapping" data-recalc-dims="1" data-src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-10/GPIO-EXTI-mapping.png?w=980&amp;ssl=1" class="lazyload" src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==">

 <noscript>

  <img decoding="async" src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-10/GPIO-EXTI-mapping.png?w=980&amp;ssl=1" alt="GPIO to EXTI mapping" data-recalc-dims="1">

 </noscript>

Secondly (and more confusingly) the EXTI controller only has 7 GPIO interrupt lines into the NVIC, and since there are more than 7 pins in each GPIO port (there are 16, in fact) this means that some of the pins have to <em>share</em> an interrupt. The first 5 (<code>EXTI0</code> to <code>EXTI4</code>) get their own interrupts, but 5–9 have to share the <code>EXTI9_5</code> interrupt, and 10–15 have to share the <code>EXTI15_10</code> interrupt. Here’s a picture to make things clearer (the extra number in the NVIC column is the <em>position</em> of the interrupt in the NVIC vector table):

<img decoding="async" alt="EXTI to NVIC mapping" data-recalc-dims="1" data-src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-10/EXTI-NVIC-mapping.png?w=980&amp;ssl=1" class="lazyload" src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==">

 <noscript>

  <img decoding="async" src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-10/EXTI-NVIC-mapping.png?w=980&amp;ssl=1" alt="EXTI to NVIC mapping" data-recalc-dims="1">

 </noscript>

This is all shown (along with the names, positions &amp; priorities) of all the other interrupts in your discoboard in Table 42, Section 11.3 on p321 of the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/v_media/manuals/stm32-L476G-discovery-reference-manual.pdf">discoboard reference manual</a>. Here’s a simplified version of that table which only contains the rows relevant to the EXTI controller:

<table>

 <thead>

  <tr>

   <th>position</th>

   <th>interrupt</th>

  </tr>

 </thead>

 <tbody>

  <tr>

   <td>6</td>

   <td><code>EXTI0</code></td>

  </tr>

  <tr>

   <td>7</td>

   <td><code>EXTI1</code></td>

  </tr>

  <tr>

   <td>8</td>

   <td><code>EXTI2</code></td>

  </tr>

  <tr>

   <td>9</td>

   <td><code>EXTI3</code></td>

  </tr>

  <tr>

   <td>10</td>

   <td><code>EXTI4</code></td>

  </tr>

  <tr>

   <td>23</td>

   <td><code>EXTI9_5</code></td>

  </tr>

  <tr>

   <td>40</td>

   <td><code>EXTI15_10</code></td>

  </tr>

 </tbody>

</table>

For this reason, the <code>GPIO_configure_input_pin_it</code> macro doesn’t enable the NVIC interrupt—you need to enable it yourself using the <code>NVIC_set</code> macro like so:

<pre><code class="language-ARM hljs"><span class="hljs-symbol">NVIC_set</span> ISER <span class="hljs-number">8</span></code></pre>

Which interrupt (in the NVIC) does the above line of assembly code enable?

ISER stands for <strong>I</strong>nterrupt <strong>S</strong>et <strong>E</strong>nable <strong>R</strong>egister; you can also use ISPR (set pending) or other register banks there, see the macros file for details.

Here’s an example of where this gets tricky: if you want to have interrupts on (say) pins <code>PE13</code> and <code>PE14</code> they will both trigger the same interrupt handler function <code>EXTI15_10_IRQHandler</code>, since they’re both in the 10–15 range. To deal with this, the handler function will have to check another register (the EXTI interrupt pending register) to see which pin number triggered the interrupt.

So, the full journey of a GPIO interrupt through your system is:

<ol>

 <li>an edge (rising or falling) is detected on your input GPIO pin</li>

 <li>the EXTI controller detects this (assuming the interrupt is enabled and it’s watching the right port) and raises one of the <code>EXTIn</code> interrupt lines to the NVIC</li>

 <li>if the <code>EXTIn</code> interrupt is enabled in the NVIC, your program is interrupted and the handler function (determined by the address in the vector table) is called</li>

</ol>

If you have trouble, here are a few questions to ask yourself:

<ol>

 <li>have you tried pressing the reset button before debugging?</li>

 <li>have you enabled the SYSCFG clock?</li>

 <li>have you clocked the GPIO pins?</li>

 <li>have you configured the GPIO pins as input pins?</li>

 <li>have you configured the GPIO pins to trigger an interrupt?</li>

 <li>have you configured the trigger for the interrupt (i.e. rising or falling edge)?</li>

 <li>have you enabled the appropriate EXTI interrupt in the NVIC?</li>

 <li>have you written the interrupt handler function, and is it globally visible?</li>

 <li>does your interrup handler function clear it’s pending register before it exits? (the <code>EXTI_PR_clear_pending</code> macro will probably help you out here)</li>

</ol>

<p class="push-box">Write a program where pressing the central joystick button blinks the red LED, but pressing one of the direction buttons blinks the green LED (don’t forget to enable the correct interrupts in the NVIC, and to disable the interrupt pending flag using the <code>EXTI_PR_clear_pending</code> macro before the handler function exits). Commit &amp; push your program to GitLab.

<h2 id="exercise-3">Exercise 3: click-over-the-wire</h2>

So far this lab has been a bit of an information dump, and all you did was turn on the LEDs with the joystick (which you’ve known how to do for ages). In this exercise, you’ll take your knowledge of general GPIO input and output and re-implement the click-to-blink program <em>again</em>, but this time sending the “click” signal over the wire.

Grab one of your jumper leads and connect it to your board from pin <code>PB7</code> to <code>PE13</code>. You’ll use one end as the receiver and one end as the sender—it doesn’t matter which. What you need to do in this exercise is:

<ul>

 <li>configure your output pin as a GPIO output</li>

 <li>configure your input pin as an interrupt-enabled GPIO input</li>

 <li>write your joystick <code>EXTI0</code> interrupt handler so that instead of toggling the LED directly, it toggles the value on the sender data pin (using the ODR helper macro)</li>

 <li>write another interrupt handler function (have a careful think about what should it be <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/10-connecting-discoboards/#input-with-interrupts">called</a>?) and enable it in the NVIC and make <em>that</em> handler function toggle the LED</li>

</ul>

The info in the first two exercises will help you out—there are a few gotchas, so read it carefully and ask for help if you get stuck. Once you’re done with that, add another wire between <code>PD0</code> and <code>PE14</code>—repeat the process so that pressing one of the joystick direction buttons toggles the other LED over the wire.

<p class="think-box">If you use <code>PD0</code> you’ll have to be a bit careful—since <code>EXTIO</code> is used by the center joystick button (wired to <code>PA0</code>), it’s NOT possible to set <code>EXTIO</code> as an interrupt for <code>PD0</code>. Why can’t we use the same handler for both A0 and D0 interrupts?

<p class="push-box">Write a program where you can toggle the red &amp; green LEDs using the joystick with the signals travelling over the wires (as described above). Commit &amp; push your program to GitLab.

<p class="extension-box">What happens if you experiment with different triggering schemes—rising+falling edge vs rising edge only? Do the triggering schemes have to be the same on both ends of the wire? How many different ways can you configure the wires &amp; interrupts in your click-over-the-wire program?

<h2 id="exercise-4">Exercise 4: <em>“multi-player”</em> QuickClick</h2>

<p class="info-box">Due to current circumstances and #socialdistancing the original version of this exercise won’t really work. So here is an altered version that only involves a single discoboard.

Remember the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/09-input-through-interrupts/#quickclick">QuickClick game</a> you made last week? In Exercise 4 you will wire up both sides of your discoboard to build a multi-player<sup id="fnref:multi" role="doc-noteref"><a class="footnote acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/10-connecting-discoboards/#fn:multi">3</a></sup> variation of QuickClick.

<p class="think-box">In this instance we are going to pretend that each side of the board is actually two separate boards, this is a bit different to actually having to separate boards but the concept is similar, if you did actually have 2 boards, then how would this be possible?

As you (hopefully) just figured out, the multiplayer part of this is super-easy because you already did all the hard work in <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/10-connecting-discoboards/#exercise-3">Exercise 3</a>. Once you’re sending a signal over a wire, it doesn’t matter whether both ends of the wire are connected to the same discoboard or <em>different</em> discoboards.

There is one caveat here: voltages, such as the “low” and “high” voltages you’ve been setting and reading from your GPIO pins, are actually <em>relative</em> measurements. Think of it like the concepts of shortness/tallness: someone might say you’re either tall or short depending on who you’re standing next to. Even though you <em>might</em> say that a person is tall (or short) what you really mean is that that person is taller (or shorter) than the average of the heights of all the other people you know. Well, voltage is similar—what you care about is the voltage <em>difference</em> between two places. When your wire is connected to your board only, there’s no problem—both ends have the same common reference or <strong>ground</strong>. But when two boards are connected to each other they need some way of agreeing on what the <code>0</code> voltage level is. This is achieved by connecting their <code>GND</code> <strong>ground</strong> reference pins together with a separate cable. Your discoboard has several <code>GND</code> pins because needing to connect boards to a common ground is so common, so you need to have plenty of pins handy to do it.

To get back to the <em>“multiplayer”</em> QuickClick exercise, what you’ll need to do is:

<ol>

 <li>Reconfigure you’re interrupts and handlers so that each side of the boards have a button on the joystick.

  <ul>

   <li>The left joystick button will be the left side’s control (wired to <code>PA1</code>)</li>

   <li>The right joystick button will be the right side’s control (wired to <code>PA2</code>)</li>

  </ul></li>

 <li>configure sender and receiver pins and interrupts for both sides of the board

  <ul>

   <li>left:

    <ul>

     <li>Receiver: <code>PB7</code></li>

     <li>Sender: <code>PD0</code></li>

    </ul></li>

   <li>right:

    <ul>

     <li>Receiver: <code>PE14</code></li>

     <li>Sender: <code>PE13</code></li>

    </ul></li>

  </ul></li>

 <li>connect one of the <code>GND</code> pins on the left of your discoboard to one of the <code>GND</code> pins on the right of your discoboard<sup id="fnref:necessary" role="doc-noteref"><a class="footnote acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/10-connecting-discoboards/#fn:necessary">4</a></sup></li>

 <li>take two wires and connect the sender on the left (<code>PD0</code>) to the receiver on the right (<code>PE14</code>), then repeat this process from the sender on the right (<code>PE13</code>) to the receiver on the left (<code>PB7</code>).</li>

 <li>figure out how the game is going to work:

  <ul>

   <li>an option here would be to assign an LED to each side and play tennis—let’s say the red led means the ball is in the left player’s court and a green LED means it is in the right player’s court. Now, when you click one of the player’s buttons it should <em>“pass”</em> the led to the other player.

    <ul>

     <li>eg: if the green LED is currently on, then pressing the right button will turn the green LED off and send a signal over the wire to the left side that the LED is now in their court and they should turn the red LED on</li>

    </ul></li>

  </ul></li>

</ol>

It may not seem like it, but this is actually similar to what you will do in assignment 3, so do stop and analyse your current program before moving on.

<span class="kksr-muted">Rate this product</span>

The fact that the GPIO interrupts go though both the EXTI and the NVIC <em>is</em> complicated, and it also means there are several different ways of enabling/disabling/triggering these interrupts. What you’ve done in this exercise is to enable it in <strong>both</strong> places: the EXTI enabling happens in the <code>GPIO_configure_input_pin_it</code> macro, and the NVIC enabling in the <code>NVIC_set</code> macro. If you want to disable it in the EXTI, you disable the interrupt for pin <em>n</em> by <strong>clearing</strong> (setting to <code>0</code>) the <em>n</em>th bit in the <code>EXTI_IMR1</code> register (base address <code>0x40010400</code>).

To disable it in the NVIC, you <strong>set</strong> (to <code>1</code>) the correct bit (see mapping diagram above) in the <code>NVIC_ICERn</code> register (base addresses starting at <code>0xE000E180</code> for <code>NVIC_ICER0</code>, but there’s lots of them—e.g. <code>EXTI15_10</code> spills into <code>NVIC_ICER1</code> register because it’s in slot 40). As before, you can use the <code>NVIC_set</code> macro for this, just use <code>ICER</code> where you used <code>ISER</code> in <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/10-connecting-discoboards/#input-with-interrupts">Exercise 2</a>. Be careful not to use the normal load-twiddle-store approach for this, though—as discussed in the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/09-input-through-interrupts/#clear-enable-gotcha">Week 8 lab</a>.

Just to recap: to disable an interrupt in the EXTI you <em>clear</em> a bit, and to disable it in the NVIC you <em>set</em> a bit, and it’s a different bit in each case. Don’t ask me why things are so inconsistent, blame the people who designed the discoboard. The <code>macros.S</code> file should handle some of this stuff for you, but that’s the full story if you’re getting confused trying to disable/re-enable interrupts in your program.

there are a lot of moving parts here, but some things to think about are:

<ul>

 <li>how do you keep track of who’s turn it is?</li>

 <li>who starts with the ball (LED) in their court?</li>

 <li>is their implicit communication going on in your implementation?

  <ul>

   <li>or could you directly take it and use 2 boards and it would work the same?</li>

  </ul></li>

</ul>

After having thought about and answered the points above, here are some things to evaluate the <em>“correctness”</em> of your solution:

<ul>

 <li>Do you turn the LEDs off and on instead of toggling them?</li>

 <li>Where do you control the LEDs? the left side should only be changing the red LED and the right side should only be changing the green LED</li>

 <li>What happens if you try to pass the LED when it isn’t on your side?

  <ul>

   <li>How do you know if the LED is on your side?</li>

   <li>Can you make it so that each side alters their output if and only if the LED is on their side?</li>

  </ul></li>

 <li>Pull out a connection and play the game, does it behave as you expect it to?</li>

 <li>Do the left and right sides share any memory locations? If they do then consider the case where they are on two separate boards, would it still function the same?</li>