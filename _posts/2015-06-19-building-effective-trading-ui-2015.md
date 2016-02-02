---
layout: post
title:  "Building Effective Trading GUI"
date:   2015-06-19 00:00:00 +0000
categories: coding javascript UI
---
### **Use Javascript.**

Same with any industry other than FinTech, we've all been there: *GDI, WinForms, Swing, Flash, WPF/Silverlight, Air*. Today that most shining crown jewel, is Javascript. Its performance, its ease of development, ocean of well and badly written packages with full source code, are the result of years of named companies throwing shed load of money and army of programmers at it, vastly skilled and unskilled developers or hobbists, pushing it to be what it became today. Along with rise of cloud operating systems like ChromeOS, matured web technologies has became a defacto platform for desktop application development. So this blog is about javascript and HTML, but I think it is not an alternative, it is where to begin with in GUI building today.

### **Language before frameworks**

Before window shopping around frameworks, or become a devout believer of any one of them, Javascript is the only true friend that give you wings throughout our journey of web GUI until that next big thing. *jQuery, BackboneJs, ExtJs, AngularJs, EmberJs, ReactJs, PolymerJs..* they are all merely js code blocks manipulating DOM elements. I have seen a lot of proficient programmers new to the web UI battlefield, start by survey and chose a bloated *'Enterprise Popular'* framework, then start wrestling with hundreds pages of documentation on APIs, memorising hundreds of framework defined keywords(`$, _, 	{{ "{{ " }}, ng, #, $scope`), and then concluded that Javascript is shit. It is not. A fit to purpose framework get us to half way at best, when framework is against us, just find a way to communicate to that DIV or Span and write JavaScript then decorates with CSS. Somewhat in reverse to server side programming, UI whips upstream multi-casted mess into shape and lines to end up in the place with bags of pixels on screen for user to interface with, anything more or anywhere else is meaningless.

- Javascript the language is more functional than OOP, experience in traditional OOP language hardly helps, many FinTech application programmers get tricked by that *Java* bit, they are apple and oranges. I would suggest if the language is holding back in early days, take a break, try some functional programming language like Clojure or Lisp, then come back it would suddenly make a lot of sense.

- Callback is about event driven data flows, overusing it for other purposes leads to hell, trying hard to avoid it leads to purgatory. Together with dynamic types, they make JavaScript effectively a great UI language if used wisely.

- It is single threaded, which is always great for UI programming, leave number crunching, multi-threaded synchronization and layers of abstractions/patterns at server side, front end programming is all about presenting content to users, passing user generated event to backbone. There is no such thing as thin client or fat client.

- Error handling. Again, as a presenter, we want our show to keep running, not panic and crash organically. I really enjoy the beauty of callback `function(err, result) { .. }`. It is like asking: do you want to display the result? when no error? or present the error? It is not in UI's gene to throw this throw that, DomainUnhandledException, logger.Fatal, process.exit(1), SIGTERM, kill \-9, etc.

**Re-usability in UI programming is a lie.** GUI is the closest project to business logics, which directly drives what human to see and touch, motivates feature requests, reiteration or even rewriting. Focus on *business logics* not MVVM, MVC, Flux. Most beautifully crafted reusable control can not withstand a tiny change of business requirement in one particuliar use case. UI project shape shifts and adapts, so KISS rule works best, *"Keep it simple, stupid"*. Creative UI is harder, but rewards with user productivity, work-flow efficiency and business exploration.

Trading systems requires low latency. Same rules to upstream and downstream services apply to UI, **the most efficient way to program is most likely anti-pattern**. In the end all our interfaces, classes, models and templates compiled down to GOTOs and JUMPs anyway.  When talking about low latency C/C++ is the king, because they offer full control over bare metal, a badly written C++ program can perform much worse; JavaScript also gives us freedom sometimes way too flexible, which makes program codes easy to get an edge, or very wrong.


That being said, take a look at [React](http://facebook.github.io/react/) :) Here is why.

- Fast
- Reactive
- Zen like model, invisibly helps departmentalize project without shoving tons of keywords in your brain
- Virtual DOM, code/html in single file fit into trading UI's local update intense and widgets rich nature
- Understand 2 states and 7 lifecycle methods anyone can start making things happen, real stuffs
- Plays nice with existing JavaScript components, even those classic ones before HTML5 era
- ReactTools extension to DevTools, easily inspect business data in running components
- Transpiling ES6, class keywords, lambda functions et. make codes much more readable and fun to write

### **DevTools before engineering**

![Chrome DevTools](https://developer.chrome.com/devtools/images/timeline-panel.png)

Our next true friend after JavaScript, is DevTools. For the past decade of UI crafting, I have never seen such a performant, powerful, integrated, ease-to-use development environment, and it sits quietly inside everyone's browser. With DevTools you can easily see what is actually happening in our application. They are really really good. Unfortunately they are also neglected by a LOT of fellow programmers I have met. It is far more than putting breakpoints and variable inspection. Here fancy algorithms could reveal its ugliness during run-time almost instantly from every corner. For pixel perfectionists your questions about why this happens are always several clicks away from its cause, be it in JavaScript or CSS or a combination, or a frame within complex animations. There are tons of great tutorials you could find on-line on how to effectively use DevTools's functionalities, and it is designed to be straightforward and easy to use. I would not go into details here. The next two sections are not possible without DevTools telescope on:

- Profiling for memory leaks
- Profiling for rendering performance

### **Memory is your friend, evil hides between frames**

Our goal is to render faster, therefore time-space trade-off is unavoidable and it really is not a thing. A typical trader spec workstation has at least 16 gigabytes of memory, next to a cache-rich Intel Xeon processor, not far from multiple workstation GPUs. Implementation first, optimization later, try not start with server side tricks to micromanage memory allocations or worry about GC: **When in doubt, simply cache it.** JavaScript gives us flexible dynamic object model, even better now that JSON became major encoding standard across languages, modern browsers are equipped with powerful profiling tools, use them.

Another fundamental difference between UI and server-side programming, is that we dealing with human ourselves, not bytes. Processing throughput and nanosecond level operations is crucial to computer programs, but numerical value fractional ticking between 100ms is trivial to most human brains. During busy market, some contracts can generate 3000 to 4000 updates per seconds, each quote change is crucial to maintain consistent order book from its exchanges, but there is absolutely no value to render this real-time on screen for human eyes. Below is one type of common code pattern for conflation.

{% highlight javascript linenos %}
// Processing market data provider's real-time pricing ticks
Pricefeed.Socket.on('quote',
  (quote) => {
    // Update cached pricing quote by merging fields
    self.cached_quotes[quote.source] = _.extend(self.cached_quotes[quote.source], quote);

    // Create timer if required,
    // otherwise keep caching data for next rendering timer event
    if (self.tmr_throttle === undefined) {
      self.tmr_throttle = setTimeout( () => {
        _.each(self.cached_quotes, (quote, source) => {
          // { Do Renderings ... }
        });
        // Cached information rendered
        self.cached_quotes = {};
        // Clear timeout handle
        self.tmr_throttle = undefined;
      }, 50); // Render once for conflated ticks cached during the last 50ms
    }
  });
{% endhighlight %}

Javascript's single threaded execution flow and **setTimeout** and **setInterval** (and their counterpart **clearTimeout**, **clearInterval**) can be magical to achieve various optimisations, and optimisations can be contained within that callback without re-engineering overall structure. This is particularly important so that we would not pollute our cleanly structured programs with lots of optimisation codes upfront. Each data feed is different, it is better to code for business logics then apply optimisations where profiling tools lead us.
- A clean and elegantly structured code might actually run as Titanic's like this:

![Run-time Titanic's of Inefficiency](/images/effectively-ui/runtime-titanics.png)

- optimise calltree to whip performance into needle shapes.

![Optimised code executions](/images/effectively-ui/runtime-needles.png)

The above result is achieved by simply wrapping 3 methods with caching tricks in execution monitor. My trading datagrid, based on modifications of [SlickGrid](https://github.com/mleibman/SlickGrid), has been stress tested with 500,000 live orders ticking with real-time Bloomberg pricefeed, and broker execution updates with animations, without sweat (smoothly *scrolling/filtering/grouping* while all these is happening).

![Trading grid](/images/webtrader_s2.png)

Be aware, caching could lead to memory leaks! Thankfully we have profiling tools just for this purpose. Take two snapshots and compare, DevTools reveals origination of everything created and left over between two snapshots. This is one of the reasons why I often write prototyping service programs in [Node.js](https://nodejs.org/). Sometimes we write fancy algorithms looking so good on paper by themselves, but once put into real world context they became bugs. Even for server side programming I tends to rely on DevTools to help me better understanding the task. Once I had a full picture, writing them in a static server programming language is a walk in garden, with confidence by knowing they can only run better. In fact, sometimes prototype stays as it is because they already out-deliver requirements. Below are some supporting daemons written in JavaScript, monitoring real-time event streams, emitting global time-zoned clock events with geographical day light savings. They have been running for more than two months without a single byte of memory leaks and still crunching thousands of events and spawning sand-boxed rule scripts 24 x 7. I have learned a lot about all the event streams from other systems while writing these "prototypes" with great help from DevTools for remote debugging.

![Long Running NodeJs Server Side Javascript Services](/images/effectively-ui/long-live-nodejs.png)


### **V8 is your friend, vendor library is not**

When vendor library is working against us, don't be obsessed with fighting back 1 on 1. Polyfilling prototypes, hijacking event listeners always comes with potential catastrophic avalanches in complex UI projects at some point. Remember we all share a simple goal, that is to update a DOM element. Take below as example, in a recent client's requirement for an intra-day exposure monitor. [KendoUI's flexible TreeList](http://demos.telerik.com/kendo-ui/treelist/index) has been used here for plotting portfolio hierarchies.

Exposure monitor's first prototype suffers from rendering hiccups. At the time of UI prototyping back-end service has not been done, this UI page needs to process around 500 orders with more than 16000 executions to aggregate P&L + Risk calculations on every aggregation levels, for every pricing or partial fill update. With some optimisation tricks calculations are controlled well below 1ms (in red circle, which is our `aggregatePosition()` function). Damn V8 is so fast it almost feels like cheating.

Profiling results show slowness came from KendoUI, everything after red circle belongs to `kendo.*`, triggered by a single call of `kendogrid.pushUpdate()`. There is no official API on BeginUpdate/EndUpdate for the grid we are using.
![Optimising DOM Update](/images/effectively-ui/devtools.png)

There are two ways to solve this, either we can look into KendoUI's function call from this graph, and bypass or modify some unnecessary calls during frame rendering. This is usually called [Polyfilling](https://remysharp.com/2010/10/08/what-is-a-polyfill). Or, for some fast ticking columns like MarketDepth, BuyNotional or SellNotional, do we really need to refresh the whole data source for them? We can directly manipulate DOM for these cells:
{% highlight javascript linenos %}
// Column definitions
{
    name: "Last",
    field: "pricefeed.LAST_PRICE",
    width: 60,
    cssClass: 'cell-align-right',
    formatter: function(..., order) {
        var html = '';
        html += "<span class='px_" + (order["symbol"]) + "lastPrice'>" +
                self.formatNumeric(prices['lastPrice']) + "</span>";
        html += "<br/><span class='footnote px_" + (order["symbol"]) + "sizeLastPrice'>"
                + self.formatNumeric(prices['sizeLastPrice']) + "</span>";
        return html;
    }
},     

...

// Pricefeed event handler update DOM content by class selector, FAST!
tick : (quote, source) => {
    _.each(quote, (val, key) => {
        $('.px_' + source.symbol + key).html(val);
    });
}

{% endhighlight %}

By offloading heavy burden on pricing rendering, our profiling now looks like this:
![Optimising DOM Update 2](/images/effectively-ui/devtools2.png)

And those market data cells are ticking directly at DOM object:
<p align="center">
<img src="/images/effectively-ui/DOMUpdate.gif" alt="Updating DOM Directly" style="width: 550px;"/>
</p>

Similar techniques are also applied to some of the heaviest updating widgets, like DMA trading widgets.

![IPad Trading](/images/effectively-ui/webtrader_s3.png)

or for mobile platforms:

![IPad Trading](/images/effectively-ui/ipad.png)

Try not to be drowned into frameworks and third-party components, trust our fellow browser programmers at Chromium or Firefox, they know what they are dealing with. Leave it to the professionals.

### **Subtle animations, contextual feedbacks**

With great power comes great responsibility. Modern GPU accelerated CSS styling does not mean it should be overused everywhere. I have seen enough *'modern'* trading UI filled with gigantic gradients, shadows, transparent glasses and in-your-face animations that make no sense at all. There is one and only one reason any animation should be used: **it provides contextual meaning, to inform the user and to imbue the app with a moment of wonder and a sense of superb craftsmanship**. Our users spend time with trading UI on a day to day basis, animations and colour palette almost always have to be subtle.

- **Neutral Colour Scheme** provides enough visual stimulation to keep the audience interested, while still allowing them enough mental freedom. Mac OS X (pre-Yosemite) is the perfect example.

- **Visual Continuity**: motion describes spatial relationships, functionality, and intention with fluidity
<p align="center">
<img src="/images/effectively-ui/bidaskanim.gif" alt="Drawing" style="width: 150px;"/>
</p>

- Responsive interaction encourages deeper exploration of an app by creating timely, logical, and delightful screen reactions to user input. Each interaction is thoughtful, perhaps whimsical, but **never distracting**.
<p align="center">
<img src="/images/effectively-ui/fillanim.gif" alt="Drawing" style="width: 450px;"/>
</p>
