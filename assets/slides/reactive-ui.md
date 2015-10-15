class: middle
# Reactive UI on Android with Scala #

<br/><br/><br/><br/><br/>
<img src="/assets/img/signals/wire.png" style="float: left; width: 96px; margin-right: 20px;"/>
<br/>
*Zbigniew Szymanski*</br>
[http://zbsz.github.io](http://zbsz.github.io)</br></br>


.footnote[.right[2015.mobilization.pl]]

---
## It's not about: ##

- Scala language
- using Scala on Android
- app design / architecture

--
<br/>

## But ... ##

- creating reactive UI
- using techniques enabled by Scala - Signals
- introduce Signals
- UI case study

---
## Signals

<br/><br/><br/>
<center><img src="/assets/img/signals/signals.png" /></center>

???
- value changing over time
- idea borrowed from FRP
- adapted to Android development

---
background-image: url(/assets/img/signals/observable.gif)
class: bg-left-top

## Observable

<br/><br/><br/><br/> 
<br/><br/><br/><br/> 


```scala
trait Signal[T] {
  
  def set(value: T): Unit
  
  def subscribe(f: T => Unit): Unit
  
}
```

???
In general, Signal is just a value that changes over time, and provides change notifications, a bit like `Observable`.
Very simplified signal interface provides methods to set and observe current value, it could look something like this:

---
## Cell in spreadsheet

<br/><br/>
<center><img src="/assets/img/signals/cells.png" style="width: 750px" /></center>


???
- signals are composable
- like cells in spreadsheet

---
## Signal != Event Stream
<br/>
<center><img src="/assets/img/signals/stream.png" /></center>

- signal contains a value
- can skip changes
- can report same state multiple times
- focus on final state, not intermediate changes


---
class: middle
# Signal Properties


---
## Empty

<br/><br/><br/>
<center><img src="/assets/img/signals/empty1.png" /></center>

???
- signals can be empty
- usually means that state is not yet loaded/computes

---
## Non-contiguous

<br/><br/><br/>
<center><img src="/assets/img/signals/empty2.png" /></center>

???
- can even have holes / disappear

---
## Skipping states

<br/><br/><br/>
<center><img src="/assets/img/signals/skipping.png" /></center>

???
- signals can skip intermediate states
- can even have holes

---
## Lazy

<br/><br/><br/>
<center><img src="/assets/img/signals/lazy.jpg" /></center>

???
- signals are lazy
- not processing when there are no subscribers 
- not processing when context is paused
- no notifications when value is unchanged

---
## Eventually Consistent

<br/><br/><br/>
<center><img src="/assets/img/signals/graph.svg" /></center>


???
- signals graph is guaranteed to be eventually consistent
- at any given points signals can be in intermediate states which may not 'add up'
- but eventually all values will align


---
# EventContext

<br/>
- corresponds to lifecycle context
  - global
  - service
  - activity
  - fragment
  - view
- every subscribe is done in some context (usually implicit)
- subscribers will only be notified when context is active
- automatic unsubscribe when context is destroyed

---
# Thread-safe API

<br/>
```scala
signal ! Value(...)
```

--

```scala
signal mutate (_ + 10) // atomic
```

--

```scala
signal.currentValue() : Option[T]
```

--

```scala
signal.on(Threading.Ui) { value => ... }
```

--

```scala
val fieldSignal = signal map (_.field)
```

--

```scala
val composedSignal = signal flatMap service.signalFromValue
```


---
class: center, middle
# Case Study #


---
# Ui Design #

.center[![Playback control ui design](/assets/img/reactive_ui/player.png)]

--

- Display current state, refresh on every change.
  - update when song is changed.
  - update when song info is changed (number of songs).
  - react to player state changes.
- What if no song is selected?
- What if album info is not loaded yet?

---
# PlaybackService #

<br/><br/>
```scala
trait PlaybackService {
  // Currently played song  
  val currentSong: Signal[Uri]

  val playbackState: Signal[PlaybackState]

  def playbackPosition(song: Uri): Signal[Millis]

  def song(uri: Uri): Signal[Song]
  def album(id: String): Signal[Album]

  def play(song: Uri): Unit
  def pause(): Unit
  def seek(position: Millis): Unit
}
```

???

Our UI needs service from which we can access album information, and control playback. 

In real world this would be couple different services,
but for the sake of our example we will just define everything in single trait. 

Api will be using Signals, if you don't know what that is, check out my [previous article]({% post_url 2015-08-28-signals %}).

---
# Quiz #

<br/><br/>
<br/><br/>
```scala
class PlaybackView {



    
  // How many lines of code do we need ?
  
  
  
}
```


---
class: wide
# Implementation #

<br/>

```scala
class PlaybackView(context: Context, attrs: AttributeSet, style: Int) 
    extends RelativeLayout(context, attrs, style) with ViewHelper {

  val service = inject[PlaybackService]               // inject our service

  inflate(R.layout.playback_view, this)               // inflate xml layout

  val tvPosition  : TextView = R.id.tvPosition        // bind text views 
  val tvDuration  : TextView = R.id.tvDuration
  val tvSong      : TextView = R.id.tvSong

  val pbProgress  : ProgressBar = R.id.pbProgress     // bind progress bar

  val btnPrevious : ImageView = R.id.btnPrevious      // bind buttons
  val btnRewind   : ImageView = R.id.btnRewind
  val btnPlay     : ImageView = R.id.btnPlay
  val btnForward  : ImageView = R.id.btnForward
  val btnNext     : ImageView = R.id.btnNext

  // rest of the code goes here ...
}
```

---
### Display current state

.center[![Playback control ui design](/assets/img/reactive_ui/display_state.png)]

--

```scala
val currentSong = service.currentSong flatMap service.song
```
--

```scala
val currentAlbum = currentSong flatMap { song =>
  service.album(song.album)
}
```

--
```scala
val currentPosition = 
  service.currentSong flatMap service.playbackPosition
```

---
### Current song label

.center[![Playback control ui design](/assets/img/reactive_ui/chapter_label.png)]

--
```scala
val songLabel = for {
  song  <- currentSong
  album <- service.album(song.album)
} yield s"Song ${album.indexOf(song) + 1} of ${album.songs.size}"
```

--

```scala
songLabel.on(Threading.ui) { tvSong.setText }
```

---

### Song duration

.center[![Playback control ui design](/assets/img/reactive_ui/duration.png)]

--

```scala
currentSong
  .map(_.duration.toSeconds.toString)
  .on(Threading.ui) { tvDuration.setText }
```

---
### Current position

.center[![Playback control ui design](/assets/img/reactive_ui/position.png)]

--

```scala
val positionSeconds = currentPosition.map(_.toSeconds)

positionSeconds
  .map(_.toString)
  .on(Threading.ui) { tvPosition.setText }
```

--

```scala
val progress = for {
  song     <- currentSong
  position <- positionSeconds
} yield position.seconds * 10000 / song.duration.seconds

progress.on(Threading.ui) { pbProgress.setProgress }
```

---
# Buttons

.center[![Playback control ui design](/assets/img/reactive_ui/buttons.png)]

<br/><br/>

- Some buttons need to be disabled
- Play button handles play and pause actions
- Execute actions on click

---
### Disable buttons

```scala
val indexInAlbum = for {
  album <- currentAlbum
  song <- currentSong
} yield (album, album.indexOf(song))
```

--

```scala
val prevSong = indexInAlbum map { case (album, index) =>
  if (index <= 0) None 
  else Some(album.songs(index - 1))
}
```
--

```scala
val nextSong = indexInAlbum map { case (album, index) =>
  if (index >= album.songs.size - 1) None 
  else Some(album.songs(index + 1))
}
```

--

```scala
prevSong.map(_.isDefined).on(Threading.ui) { 
  btnPrevious.setEnabled 
}
nextSong.map(_.isDefined).on(Threading.ui) { 
  btnNext.setEnabled 
}
```

---
### Play button icon

.center[![Playback control ui design](/assets/img/reactive_ui/play.png)]

--

```scala
service.playbackState
  .map { state =>
    if (state.isPlaying) R.drawable.ico_pause 
    else R.drawable.ico_play 
  }.on(Threading.ui) { 
    btnPlay.setImageResource 
  }
```

---
class: wide
### Execute commands

```scala
btnPlay setOnClickListener { v: View =>
  if (service.playbackState.currentValue.exists(_.isPlaying)) service.pause()
  else service.currentSong.currentValue foreach service.play
}
```

--

```scala
btnRewind setOnClickListener { v: View => 
  service.seek(currentPosition.currentValue.fold(Millis(0))(_ - Seconds(30))) 
}

btnForward setOnClickListener { v: View => 
  service.seek(currentPosition.currentValue.fold(Millis(0))(_ + Seconds(30))) 
}
```

--

```scala
btnPrevious setOnClickListener { v: View => 
  prevSong.currentValue.flatten foreach { song => service.play(song.uri) } 
}

btnNext setOnClickListener { v: View => 
  nextSong.currentValue.flatten foreach { song => service.play(song.uri) } 
}
```

---
# ProgressBar Input

.center[![Playback control ui design](/assets/img/reactive_ui/progress_bar.png)]

- ProgressBar is not great
- Let's use horizontal swipe gesture on whole view


---
### Touch Helpers

<br/><br/>

```scala
val reactor = new TouchReactor(this)
```

--

```scala
val dragGesture = new DragGesture(getContext, reactor)
```

--

```scala
def swipeDistance(startX: Float) = 
    Signal.lift(dragGesture.onDrag map { 
      case (x, _) => (x - startX) / getWidth 
    })
```
--

```scala
def duration = currentDuration.currentValue.fold(0)(_.toMillis)
```

---
class: wide
### Swipe Gesture

<br/>
```scala
val swipePosition: Signal[Option[Millis]] = {

  val swipeStart = Signal(Option.empty[Float])

  dragGesture.onDragStart { case (x, _) => swipeStart ! Some(x) }

  dragGesture.onDragEnd { dragged =>
    if (dragged) swipePosition.currentValue.flatten foreach service.seek
    swipeStart ! None
  }

  swipeStart flatMap {
    case None => Signal.const(Option.empty[Millis])
    case Some(startX) =>
      val startPos = currentPosition.currentValue.getOrElse(Millis(0))
      swipeDistance(startX) map { diff => Some(startPos + duration * diff) }
  }
}
```

---
### Update UI while swiping

<br/><br/><br/>

```scala
val positionSeconds = currentPosition.map(_.toSeconds)

positionSeconds
  .map(_.toString)
  .on(Threading.ui) { tvPosition.setText }
```

--

```scala
val positionSeconds = swipePosition flatMap {
  case Some(position) => Signal.const(position.toSeconds)
  case None           => currentPosition.map(_.toSeconds)
}
```

---
class: center, middle
<img src="/assets/img/signals/demo.png" />

---
class: center
<img src="/assets/img/signals/github.png" />
<br/><br/>
<br/><br/>

## https://github.com/zbsz/reactive-ui-sample



---
class: center, middle
<img src="/assets/img/signals/thanks.jpg" />

