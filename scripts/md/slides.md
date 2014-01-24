title: Overview
class: big
build_lists: false

Topics:

- what a Run Loop is?
- Why a Run Loop
- Run Loop Queues in Ember
- autoRun
- autoRun and Testing
- Other methods of Ember.run

---

title: what a Run Loop is?
subtitle: 
class: segue dark nobackground

---

title: what?

... not a `loop` like programmers might think

<pre class='prettyprint' data-lang='javascript'>
for (i = 0; i < length; i++) {
  // some stuff here
}
</pre>

It is an 'aggregate changes over time' mechanism.

---

title: what?

<q>..used to batch, and order (or reorder) work in a way that is most ... efficient ... by scheduling work on specific queues. These queues have a priority,
and are processed to completion in priority order.</q>

<footer class='source'><a href='https://code.stypi.com/stefanpenner/se%C3%B1or%20runloop'>Steffan Penner</a></footer>

---

title: what?
subtitle: Queues of work to be done

![Queues are like buckets](images/bucket.png)
![Queues are like buckets](images/bucket.png)
![Queues are like buckets](images/bucket.png)
![Queues are like buckets](images/bucket.png)
![Queues are like buckets](images/bucket.png)
![Queues are like buckets](images/bucket.png)

Queues are run in order of priority

---

title: Why a Run Loop?
class: segue dark nobackground

---

title: why?
content_class: flexbox hcenter vcenter

Some things 'cost' more than others

- DOM manipulation
- layout calculations
- your cool, but expensive algorithm

---

title: why?

Ineffective use of DOM calculations / layout
<pre class='prettyprint' data-lang='javascript'>

// assuming divs foo, bar, baz
foo.style.height = "500px";
foo.offsetHeight;

bar.style.height = "400px";
foo.offsetHeight;

baz.style.height = "200px";
foo.offsetHeight;
</pre>

<footer class='source white'><a href=''>Steffan Penner</a></footer>

---

title: why?

Ineffective use of DOM calculations / layout
<pre class='prettyprint' data-lang='javascript'>

// assuming divs foo, bar, baz
foo.style.height = "500px"; // write
foo.offsetHeight; // read (recalculate style, layout)

bar.style.height = "400px"; // write
foo.offsetHeight; // read (recalculate style, layout)

baz.style.height = "200px"; // write
foo.offsetHeight; // read (recalculate style, layout)
</pre>

---

title: why?

Grouping writes before reads helps

<pre class='prettyprint' data-lang='javascript'>

// assuming divs foo, bar, baz
foo.style.height = "500px"; // write
bar.style.height = "400px"; // write
baz.style.height = "200px"; // write

foo.offsetHeight; // read (recalculate style, layout)
<b>bar.offsetHeight; // read (style and layout is already known)</b>
<b>baz.offsetHeight; // read (style and layout is already known)</b>

</pre>

---

title: why?

<pre class='prettyprint' data-lang='javascript'>
var Meetup = Ember.Object.extend({
  title: 'Ember.js',
  location: 'SLC',
  food: 'pizza',
  prettyTitle: function() {
    return this.get('title') + ' - ' + this.get('location');
  }.property('title', 'location');
});
</pre>

<pre class='prettyprint' data-lang='handlebars'>
  {{location}}
  {{prettyTitle}}
</pre>

---

title: why?

<pre class='prettyprint' data-lang='javascript'>
var Meetup = Ember.Object.extend({
  title: 'Ember.js',
  location: 'SLC',
  food: 'pizza',
  prettyTitle: function() {
    return this.get('title') + ' - ' + this.get('location');
  }.property('title', 'location');
});
</pre>

<pre class='prettyprint' data-lang='javascript'>
  var meetup = Meetup.create();
  meetup.set('location', 'PDX');
  meetup.set('title', 'Emberenos');
</pre>

---

title: why?
subtitle: without the run loop

<pre class='prettyprint' data-lang='javascript'>
  var meetup = Meetup.create();
  meetup.set('location', 'PDX'); // {{location}} and {{prettyTitle}} updated
  meetup.set('title', 'Emberenos'); // {{title}} and {{prettyTitle}} updated
</pre>

---

title: why?
subtitle: with the run loop

<pre class='prettyprint' data-lang='javascript'>
  var meetup = Meetup.create();
  meetup.set('location', 'PDX');
  meetup.set('title', 'Emberenos');
  // {{location}}, {{title}} and {{prettyTitle}} updated
</pre>

---

title: why?
subtitle: aggregate = net zero change

<pre class='prettyprint' data-lang='javascript'>
  var meetup = Meetup.create();
  meetup.set('location', 'PDX');
  meetup.set('title', 'Emberenos');

  meetup.set('location', 'SLC');
  meetup.set('title', 'Ember.js');

  // no DOM update
</pre>

---

title: Run Loop Queues in Ember
class: segue dark nobackground

---

title: queues
class: big
build_lists: true

<pre class='prettyprint' data-lang='javascript'>
  Ember.run.queues
</pre>

- `sync               `: synchonization jobs, propagate bound data
- `actions            `: general work queue, scheduled tasks, promises, etc
- `routerTransitions `: routing transition jobs
- `render             `: jobs meant for rendering, typically will update the DOM
- `afterRender        `: after the render is complete
- `destroy            `: jobs to finish tear-down objects other jobs have destroyed

---

title: psuedo algorithm

![Queues are like buckets](images/sync_bucket.png)
![Queues are like buckets](images/actions_bucket.png)
![Queues are like buckets](images/rtrans_bucket.png)
![Queues are like buckets](images/render_bucket.png)
![Queues are like buckets](images/arender_bucket.png)
![Queues are like buckets](images/destroy_bucket.png)


---

title: psuedo algorithm
subtitle: how the run loop processes jobs
build_lists: true

-  if no queues with jobs, return -- done
-  first `QUEUE` with pending jobs = `CURRENT_QUEUE`
-  build temp `WORK_QUEUE`
-  pull `CURRENT_QUEUE` into `WORK_QUEUE`
-  process jobs one at a time in `WORK_QUEUE`
-  repeat

---

title: psuedo algorithm
subtitle: how the run loop processes jobs

![Queues are like buckets](images/sync_bucket.png)
![Queues are like buckets](images/actions_bucket.png)
![Queues are like buckets](images/rtrans_bucket.png)
![Queues are like buckets](images/render_bucket.png)
![Queues are like buckets](images/arender_bucket.png)
![Queues are like buckets](images/destroy_bucket.png)

---

title: autoRun
subtitle: 
class: segue dark nobackground

---

title: autoRun

In development and production Ember will automatically wrap code it finds in an `autoRun` when appropriate

<pre class='prettyprint' data-lang='javascript'>
  doSomething: function() {
    this.set('propA', 3);
    this.set('propB', 4);
  }
</pre>

<pre class='prettyprint' data-lang='javascript'>
  doSomething: function() {
    Ember.run.begin();
    this.set('propA', 3);
    this.set('propB', 4);
    nextTick(function(){
      Ember.run.end();
    });
  }
</pre>

---

title: autoRun
subtitle: wrap your async

<pre class='prettyprint' data-lang='javascript'>
  doSomething: function() {
    Ember.run(function() {
      this.set('propA', 3);
      this.set('propB', 4);
    });
  }
</pre>

---

title: potential situations to consider

- $.ajax callbacks
- animation complete handler
- setTimeout
- setInterval
- postMessage
- MessageChannel
- jQuery event handling
- websocket

Generally anything async that you write should consider the run loop.

---

title: autoRun and Testing
subtitle: 
class: segue dark nobackground

---

title: autoRun and Testing
content_class: flexbox vcenter hcenter

Ember testing mode disables `autoRun`

Can cause testing issues if you are unaware of `autoRun`

---

title: Other `Ember.run` methods
subtitle: 
class: segue dark nobackground

---

title: Ember.run stuff

- `Ember.run.join`
- `Ember.run.later` <- use instead of `setTimeout`
- `Ember.run.next` <- next tick, consider `schedule`
- `Ember.run.schedule` <- push to queue
- `Ember.run.throttle`

plus more in <a href='http://emberjs.com/api/classes/Ember.run.html'>Ember docs</a>


---

title: Other Resources

- <a href='http://stackoverflow.com/questions/13597869/what-is-ember-runloop-and-how-does-it-work#answer-14296339'>machty SO post</a>
- <a href='https://code.stypi.com/stefanpenner/se%C3%B1or%20runloop'>Steffan Penner - señor runloop</a>
- <a href='http://emberjs.com/api/classes/Ember.run.html'>Ember.run Docs</a>
