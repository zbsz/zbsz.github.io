class: middle
# Reactive UI on Android with Scala #

???
## October 17 ##

---
## It's not about: ##

- Scala language
- using Scala on Android
- app design / architecture

--
<br/>

## But ... ##

- creating reactive UI
  - using techniques enabled by Scala

---
class: bg-left-bottom
background-image: url(/assets/img/signals/signals.png)

## Signals

<br/><br/>
- idea borrowed from FRP
- adapted to Android development

---
## Simplified Signal

<br/><br/><br/> 
```scala
trait Signal[T] {
  def set(value: T): Unit
  def subscribe(f: T => Unit): Unit
}
```

- value changing over time
- subscribers are notified on changes

???
In general, Signal is just a value that changes over time, and provides change notifications, a bit like `Observable`.
Very simplified signal interface provides methods to set and observe current value, it could look something like this:

---
## Cell in a spreadsheet

<br/><br/><br/>
<img src="/assets/img/signals/cells.png" style="width: 750px" />

- signals are composable

???
Signals are much more then just simple observable, they are composable.
In this regard, signals can be compared to cells in a spreadsheet, where value of one cell can be defined as a function taking inputs from other cells.

---
## Signals and Reactive Streams

<br/><br/><br/>
- somewhat similar to RxJava
- signal -> event stream
- event stream -> signal
- stream -> signal -> stream !
- focus on final state, not change events

???

Signals may seem very similar to [Reactive Extensions](http://reactivex.io/), signal can be viewed as a stream of change events, 
and event stream could be converted into signal using `scan` operator.
But this comparison can be misleading, with signals, we don't really care about the events, we are only interested in current state. Actually, we 
are mostly interested in final state, after all changes are applied. This difference has significant implications in how signals can be used compared to event streams.

---
## EventContext

<br/>
- corresponds to lifecycle context
  - global
  - service
  - activity
  - fragment
  - view
- every subscribe is done in some context (usually implicit)
- subscribers will not be notified when context is not active
- automatic unsubscribe when context is destroyed

---
### Signal properties

<br/>
- subscribers will be notified with final value (eventually)
- signal is allowed to skip intermediate states
- signal is allowed to notify subscribers multiple times
  - notify on subscribe
  - notify on context resume
- subscriber can specify its execution context
- otherwise signal is processed synchronously
- signal can be empty
  - subscribers are not notified
- signals graph is eventually consistent
- signal api is thread safe

---
class: center, middle
# Case Study #

---
# Ui Design #

.center[![Playback control ui design](/assets/img/reactive_ui/player.png)]

--

- Display current state, refresh on every change.
  - Update when chapter is changed.
  - Update when book info is changed (number of chapters).
  - React to player state changes.
- What if no chapter is selected?
- What if book info is not loaded yet?

---
# PlaybackService #

```scala
trait PlaybackService {
  // Currently played chapter  
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

Our UI needs service from which we can access book information, and control playback. 

In real world this would be couple different services,
but for the sake of our example we will just define everything in single trait. 

Api will be using Signals, if you don't know what that is, check out my [previous article]({% post_url 2015-08-28-signals %}).

---
# Quiz #

```scala
class PlaybackView {



    
  // How many lines of code do we need ?
  
  
  
}
```


---
class: wide
# Implementation #


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

### Chapter duration

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

- some buttons need to be disabled
- play button handles play and pause actions
- execute action on click

---
class: wide
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
  if (index <= 0) None else Some(album.songs(index - 1))
}
```
--

```scala
val nextSong = indexInAlbum map { case (album, index) =>
  if (index >= album.songs.size - 1) None else Some(album.songs(index + 1))
}
```

--

```scala
prevSong.map(_.isDefined).on(Threading.ui) { btnPrevious.setEnabled }
nextSong.map(_.isDefined).on(Threading.ui) { btnNext.setEnabled }
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
def duration = 
  currentChapter.currentValue.fold(0)(_.duration.toMillis)
```

---
class: wide
### Drag Gesture

```scala
val dragPosition: Signal[Option[Millis]] = {

  val dragStart = Signal(Option.empty[Float])

  dragGesture.onDragStart { case (x, _) => dragStart ! Some(x) }

  dragGesture.onDragEnd { dragged =>
    if (dragged) dragPosition.currentValue.flatten foreach service.seek
    dragStart ! None
  }

  dragStart flatMap {
    case None => Signal.const(Option.empty[Millis])
    case Some(startX) =>
      val startPos = currentPosition.currentValue.getOrElse(Millis(0))
      swipeDistance(startX) map { diff => Some(startPos + duration * diff) }
  }
}
```

---
### Update UI while swiping


```scala
val positionSeconds = currentPosition.map(_.toSeconds)

positionSeconds
  .map(_.toString)
  .on(Threading.ui) { tvPosition.setText }
```

--

```scala
val positionSeconds = dragPosition flatMap {
  case Some(position) => Signal.const(position.toSeconds)
  case None           => currentPosition.map(_.toSeconds)
}
```

---
class: center, middle
# Demo


---
class: center, middle
# Thank you

- https://github.com/zbsz/reactive-ui-sample
