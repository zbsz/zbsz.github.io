---
layout: post
title: Reactive UI
tags : [scala, android, signals]
tagline: "signals case study"
---

Let's create playback control view for an audiobook app. Audiobook consists of several chapters, our view will show playback progress of current chapter,
and provide user with basic actions to control playback and switch chapters. Assuming that we already have ui layout properly done in xml, and 
playback service gives us nice api based on [Signals]({% post_url 2015-08-28-signals %}), we are going to implement view controller layer in reactive way. 

## Prerequisites

### Ui Design

Our playback control is quite straight-forward, has couple basic elements that we would expect to see in audio player. Here is what we got from a designer.

![Playback control ui design]({{ site.url }}/assets/img/reactive_ui/player.png)

Let's assume that we already have layout and all other resources prepared and ready to use in our view implementation.

### Service Interface

Our UI needs service from which we can access book information, and control playback. In real world this most likely be couple different services,
but for the sake of our example we will just define everything in single trait. Api will be using Signals, 
if you don't know what that is, check out my [previous article]({% post_url 2015-08-28-signals %}).

```scala
trait PlaybackService {
  // Currently played chapter  
  val chapterId: Signal[ChapterId]
  
  val playbackState: Signal[PlaybackState]
  
  def playbackPosition(chapter: ChapterId): Signal[Millis]
  
  def chapter(id: ChapterId): Signal[Chapter]
  def audioBook(id: BookId): Signal[AudioBook]
  
  def play(chapter: ChapterId): Unit
  def pause(): Unit
  def seek(position: Millis): Unit
}
```

We are not defining all classes used in this api, let's just assume that they contain what we need.

## Implementation

Now that we have all dependencies defined, we can start implementing our view, we will do that step by step.

### Inflate and bind views

At first let's define our custom view and bind all required parts together. Used `ViewHelper` trait provides utility methods for view 
inflation and binding, as well as dependency injection, all of that is not relevant in this example, so let's just assume that presented code works.  

```scala
class PlaybackView(context: Context, attrs: AttributeSet, style: Int) 
    extends RelativeLayout(context, attrs, style) with ViewHelper {

  val service = inject[PlaybackService]               // inject our service

  inflate(R.layout.playback_view, this)               // inflate xml layout
    
  val tvPosition  : TextView = R.id.tvPosition        // bind text views 
  val tvDuration  : TextView = R.id.tvDuration
  val tvChapter   : TextView = R.id.tvChapter
    
  val pbProgress  : ProgressBar = R.id.pbProgress     // bind progress bar
    
  val btnPrevious : ImageView = R.id.btnPrevious      // bind buttons
  val btnRewind   : ImageView = R.id.btnRewind
  val btnPlay     : ImageView = R.id.btnPlay
  val btnForward  : ImageView = R.id.btnForward
  val btnNext     : ImageView = R.id.btnNext
    
  // rest of the code goes here ...
}
```

### Display current state

Ui needs to show what chapter is currently played and current playback position, we also need to refresh it every time it changes. 
We'll start by creating helper signals which will be used in multiple places, for example we would need to access currently played chapter and currently played book.
 
```scala
val currentChapter = service.chapterId flatMap service.chapter

val currentBook = currentChapter flatMap { chapter =>
  service.audioBook(chapter.bookId)
}

val currentPosition = service.chapterId flatMap service.playbackPosition
```

That was easy, with simple `flatMap` we can extract info based on current chapter id.

#### Current chapter label

Chapter label displays an index of current chapter and count of chapter in current book as a string, eg. `Chapter 1 of 15`.
To compose this string we need to use current chapter and current book info, we will use previously defined helper signals.

```scala
val chapterLabel = for {
  chapter <- currentChapter
  book    <- service.audioBook(chapter.bookId)
} yield s"Chapter ${chapter.index + 1} of ${book.chapters.length}"
```

Once we have the label defined as signal we can actually update appropriate text view. View operations can only be done from UI thread, for that we 
need to use special `ExecutionContext`, in my code this useful thing is defined in global object `Threading.ui`. 

```scala
chapterLabel.on(Threading.ui) { tvChapter.setText }
```

This single line of code will make sure that our text view always has proper value.

#### Chapter duration

Setting chapter duration label is even easier now.

```scala
currentChapter.map(_.duration.toString).on(Threading.ui) { tvDuration.setText }
```

Notice that we used `map` to extract only required string and only then we subscribe to resulting signal.
This takes advantage of Signal _shortcutting_ ensuring that `setText` is only called
if chapter duration is changed, and ignores all other properties of chapter. 

Alternatively we could write this part like this:

```scala 
currentChapter.on(Threading.ui) { chapter => tvDuration.setText(chapter.duration.toString) }
```

But in that case `setText` might be called more often then actually needed, important thing to consider here, is that this happens on UI thread, 
which in itself is an expensive operation. We should always avoid unnecessary operations on UI thread.


#### Current position

Every time playback position changes we need to update progress bar and label. This may seem as straight-forward as updating chapter duration, but there
is one hidden catch here. Depending on audio player implementation `currentPosition` property may be updated multiple times per second, even hundreds of times.
We need to make sure that UI is not being updated so often, updating it once a second should be completely fine. Again, we can use Signal `shortcutting` to achieve that.

```scala
val positionSeconds = currentPosition.map(_.toSeconds)

positionSeconds.on(Threading.ui) { seconds => tvPosition.setText(seconds.toString) }
```

Android `ProgressBar` accepts progress as int value in range 0 - 10000 (by default), so we need to take chapter duration into account.

```scala
val progress = for {
  chapter  <- currentChapter
  position <- positionSeconds
} yield position.seconds * 10000 / chapter.duration.seconds

progress.on(Threading.ui) { pbProgress.setProgress }
```

### Buttons

#### Disable buttons

Buttons should be enabled only if action is possible, in our case we need to disable buttons switching current chapter if there is no next or previous chapter.
We need to first create signals with this information, we will get next and previous chapter ids at the same time, since we will need them later.

```scala
val chapterWithBook = currentChapter zip currentBook // it's always good to reuse common parts

val prevChapter = chapterWithBook map { 
  case (chapter, book) if chapter.index == 0 => None
  case (chapter, book) => Some(book.chapters(chapter.index - 1)
}

val nextChapter = chapterWithBook map { 
  case (chapter, book) if chapter.index >= book.chapters.length - 1 => None
  case (chapter, book) => Some(book.chapters(chapter.index + 1)
}

prevChapter.map(_.isDefined).on(Threading.ui) { btnPrevious.setEnabled }
nextChapter.map(_.isDefined).on(Threading.ui) { btnNext.setEnabled }
```
#### Play button icon

Play button handles both play and pause actions, so its icon has to be changed based on current state.

```scala
service.playbackState.map(_.isPlaying).on(Threading.ui) { playing =>
  btnPlay.setImageResource(if (playing) R.drawable.ico_pause else R.drawable.ico_play)
}
```

#### Execute commands

Button actions depend on current state, for that Signals have an api to access state directly.
This may be unsafe, we should only access value of signals which have registered subscribers, otherwise their value may not be up to date. We also need to think
about threading, Signal api is thread safe, but it doesn't guarantee that the value we got wasn't just changed by other thread.
In our case, none of this should be a problem, in worse case we will call `play()` or `pause()` multiple times, and our service should handle that.

```scala
btnPlay setOnClickListener { v: View =>
  if (service.playbackState.currentValue.exists(_.isPlaying)) service.pause()
  else service.chapterId.currentValue foreach service.play
}

btnRewind setOnClickListener { v: View => 
  service.seek(currentPosition.currentValue.fold(0)(_ - 30.seconds) 
}

btnForward setOnClickListener { v: View => 
  service.seek(currentPosition.currentValue.fold(0)(_ + 30.seconds)
}

btnPrevious setOnClickListener { v: View =>
  prevChapter.currentValue.flatten foreach service.play
}

btnNext setOnClickListener { v: View =>
  nextChapter.currentValue.flatten foreach service.play
}
```

### Drag Gesture

The last thing to implement is allowing user to select arbitrary playback position by dragging the progress bar. 

We could handle that with `ProgressBar` change callbacks, would just need to make sure to distinguish user input generated changes with our changes from service, 
and maybe stop updating progress bar when user is touching it.

But standard progress bar is not very nice to use, it's also not accurate enough bor our needs. It's better to handle drag gesture on whole player control view
instead, this way user can just swipe anywhere around progress bar to move it. 

Handling gestures in reactive fashion deserves its own series of articles, we will not go into details here, let's just use couple helper classes and assume 
that they do what we actually need. 

```scala
val reactor = new TouchReactor(this)
val dragGesture = new DragGesture(getContext, reactor)

def duration = currentChapter.currentValue.fold(0)(_.duration.toMillis)

def swipeDistance(startX: Float) = 
    Signal.wrap(dragGesture.onDrag map { 
      case (x, _) => (x - startX) / getWidth 
    })
```

`TouchReactor` is our generic `OnTouchListener`, it transforms standard touch events into reactive `EventStream`s. This allows us to implement touch handling
as several, mostly independent gestures. `DragGesture` consumes touch events from reactor and produces events related to dragging. 

With this helpers in place, we can define drag handling logic.

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
      val startPos = currentPosition.currentValue.getOrElse(0)
      swipeDistance(startX) map { diff => Some(startPos + duration * diff) }
  }
}
```

This code actually does two main things, it calls `service.seek` at the end of drag gesture, and provides signal with current drag
position, we can use this signal to update UI to give user instant feedback during swipe.

To have UI updates during swipe gesture we just need to rewrite `positionSeconds` signal to take user actions into account.

```scala
val positionSeconds = dragPosition flatMap {
  case Some(position) => Signal.const(position.toSeconds)
  case None           => currentPosition.map(_.toSeconds)
}
```

Done, now our UI reacts nicely to user input and updates from player service. 
We implemented complete control in around 100 lines of code, and it handles all possible interactions with any special case handling, all thanks to Signals and 
reactive service api.
