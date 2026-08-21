# sdl3

SDL3 for sysl — a window, an accelerated renderer, the event queue, keyboard, mouse and touch, the
clipboard, the system file dialog, queued audio, and the monotonic clock a frame loop is paced by.
It binds what a phone needs as well as what a desktop does: the safe area, the display scale, the
on-screen keyboard, and the lifecycle events an app is stopped and restarted by.

```
dependencies {
  sdl3 { git = "github.com/sysl-lang/sdl3", version = "0.2.6" }
}
```

```sysl
import sh.sysl.sdl3.*
import sh.sysl.sdl3.c.{INIT_VIDEO, WINDOW_RESIZABLE}

main()
    init(INIT_VIDEO)

    val window = create_window("hello", 640, 480, WINDOW_RESIZABLE).expect("a window")
    val renderer = window.create_renderer().expect("a renderer")

    var running = true

    while running
        while true
            poll_event() match
                Some(e) -> if e.kind() == EventKind.Quit then running = false
                None -> break

        renderer.clear_to(rgb(20, 20, 30))
        renderer.fill_circle(320.0, 240.0, 80.0, rgb(220, 80, 60))
        renderer.present()
```

There is no teardown. The renderer and the window go when the last reference to each does, and the
process ending does the rest.

## Two layers, and why a program may reach into the lower one

`sh.sysl.sdl3.c` holds everything that is C: the link directive, the headers, the blocks that ask
the C compiler for SDL's numbers, the six opaque handles, the ABI structs and the hundred and
thirty declarations. `sh.sysl.sdl3` is what an application imports.

The split keeps two jobs apart. The lower one has to be **faithful** — a signature that disagrees
with the header links perfectly and corrupts the call at run time — and the upper one has to be
**pleasant**, which is a different question and would otherwise be answered in the same breath.

**SDL's masks and scancodes stay in `c`, and a program names the ones it wants.** They are an index
space rather than a small closed set — `KeyboardState` is literally an array indexed by scancode,
and SDL has two hundred and forty of them — so there is nothing for the upper layer to improve
about them:

```sysl
import sh.sysl.sdl3.*
import sh.sysl.sdl3.c.{WINDOW_RESIZABLE, SCANCODE_ESCAPE, KMOD_SHIFT}
```

`c.Rect`, `c.FRect`, `c.FPoint` and `c.Vertex` live there for the same reason — they are SDL's own
layouts and cross by pointer. `rect`, `frect` and `fpoint` build one without naming the type.

## The numbers are asked for, not written down

Every constant this package uses is a `c const` block: the C compiler works the value out from
SDL's own headers, for the target being built. `design/15 §7` calls a transcribed constant "correct
on one machine" with "nothing checking it".

**Converting the 262 that were transcribed found two that were wrong.** Neither was anything a test
here could have caught, because the number it would have compared against is the number that was
wrong:

| written down as | which is actually | what happened |
|---|---|---|
| `WINDOW_ALWAYS_ON_TOP = 0x8000` | `SDL_WINDOW_MOUSE_RELATIVE_MODE` | a window asking to stay above the others had the pointer hidden and warped instead |
| `EVENT_TEXT_EDITING = 0x304` | `SDL_EVENT_KEYMAP_CHANGED` | an input method's composition events never matched anything |

Both compiled, linked and ran.

## Installing SDL3

Nothing is vendored here. SDL is a large C project with its own build system and its own platform
backends, so a package carrying it would be maintaining a port rather than binding a library.

```
brew install sdl3                     # macOS
sudo apt install libsdl3-dev          # Debian / Ubuntu
```

**The headers are needed as well as the library**, because of the section above. Neither is something
you name:

```
sysl run prog.sysl
```

`package.hocon` declares `requires { pkg_config { sdl3 = … } }`, so the compiler asks pkg-config for
both ends at once. Without SDL3 installed the refusal names SDL3 and says how to install it, rather
than clang reporting a file the caller never wrote.

Until 0.2.1 this took two flags, and the include one had to know that the path to give is the
directory *above* `SDL3` — which is exactly the sort of thing pkg-config knows and a reader should
not have to.

**The prefix still cannot live in this file**, and that has not changed: `design/15 §8` refuses both a
`@link_path` attribute and a `package.hocon` field for one, because where a prefix lives is a fact
about somebody's laptop rather than a property of the package. What changed is who answers the
question. `--include-path sdl3=<dir>` and `--link-path <dir>` still answer it and take precedence, and
`LIBRARY_PATH` and `CPATH` work too since clang reads them. **Needs sysl 0.0.56.**

## The companion libraries are separate packages

| package | library | what it adds |
|---|---|---|
| [`sdl3`](https://github.com/sysl-lang/sdl3) | `SDL3` | this one |
| [`sdl3-ttf`](https://github.com/sysl-lang/sdl3-ttf) | `SDL3_ttf` | text rendered from a font file |
| [`sdl3-image`](https://github.com/sysl-lang/sdl3-image) | `SDL3_image` | PNG, JPEG and the rest, decoded |
| [`sdl3-mixer`](https://github.com/sysl-lang/sdl3-mixer) | `SDL3_mixer` | mixing, looping and fading sound |

**That split is forced rather than chosen.** A link directive is never pruned — every unit of a
compilation contributes its libraries, whether or not the program reaches it — so one package
holding all four would put `-lSDL3_ttf -lSDL3_image -lSDL3_mixer` on the link line of every program
that used any of it, and a machine with only SDL3 installed could not link a program that draws
rectangles. Depend on what you use.

## Handles own what they point at

`Window`, `Renderer`, `Texture`, `Surface` and `AudioStream` are each reached through a `&T` with an
`impl Drop`, so the C object goes when the last reference to it does. **`destroy` is not part of
this API.** A window closed twice, or drawn into after it was closed, is no longer something a
program can express.

**Anything that creates one answers `Option`.** SDL says "failed" with a null pointer and the reason
in `error()`; an `Option` is the same information in the shape the language already reads, and a
null handle cannot be passed on by accident.

**Everything else answers `bool`, exactly as SDL does.** A binding that invented a richer failure
type than the library gives would be inventing the failure cases too. When something answers
`false`, `error()` says why.

**`quit()` frees everything SDL made, so nothing may still be holding a handle when it is called.**
That is SDL's rule rather than this binding's, and a destructor cannot know about it. In practice a
program that never calls `quit()` is fine — the process ending does the same work — and one that
does should let its handles go out of scope first.

### Cursors are two types, because SDL gives no other way to say it

`create_system_cursor` answers a `&Cursor` this program owns. `current_cursor` and `default_cursor`
answer a `BorrowedCursor`, which can be made active and **has no destructor at all**.

Cairo's binding solves the same problem with one type, by taking a reference on anything the library
lends. SDL has no `SDL_ReferenceCursor`, so the honest answer is that a lent cursor is a different
thing — and a type says it in a form the compiler reads, where the previous version of this package
said it in a comment beginning *"never for what `default_cursor` answers"*.

## Enumerations rather than piles of constants

Thirteen of them: `EventKind`, `PixelFormat`, `Colorspace`, `BlendMode`, `TextureAccess`,
`ScaleMode`, `MouseButton`, `AudioFormat`, `SystemCursor`, `FileDialogKind`, `TextInputType`,
`Capitalization` and `SystemTheme`. Each has a `code` for going out
to SDL; the ones SDL ever *answers* also have an `of` and an `Other` arm for a value this package
does not name, which is an ordinary answer rather than a failure.

```sysl
poll_event() match
    Some(e) ->
        e.kind() match
            Quit -> running = false
            KeyDown -> if e.key_scancode() == SCANCODE_ESCAPE then running = false
            MouseButtonDown -> if e.mouse_button() == MouseButton.Left then click(e.mouse_x(), e.mouse_y())
            _ -> ()
    None -> ()
```

Each also has a `Display`, so a report says `window pixel size changed` rather than `519`.

## Callbacks

SDL takes a C function pointer, and a `*extern` cannot capture. So the closure APIs here —
`add_event_watch`, `show_file_dialog` — keep the closure in module storage and hand SDL a single
static trampoline that finds it again.

**A closure captures a plain local by copy** (`design/12 §7`), so a watch that counted into a `var`
beside it would count into its own field and the caller would see nothing. Capturing a `&T` retains
it, and a write through the reference reaches the one object:

```sysl
struct Tally
    n: int

val t: &Tally = Tally(0)

add_event_watch(e -> if e.kind() == EventKind.Quit then t.n += 1)
```

## The clock

`ticks()` and `ticks_ns()` count from `init`, and `performance_counter()`/`performance_frequency()`
are the raw pair behind them. **This is the only clock a sysl program has** — the language has no
time source of its own — so a frame loop measures its step with it:

```sysl
val now = ticks_ns()
val dt = real(now - last) / 1e9        // seconds, for the physics
last = now
```

It is **monotonic**: it counts forward from `init` and does not jump when somebody changes the
system time, which is what makes it right for a frame time and useless for a date. `delay_precise`
is the companion for pacing — it spins out the last of the interval rather than handing the whole of
it to the scheduler, and costs a core while it waits.

## The safe area, which is the whole window until it is not

`window.safe_area()` answers the part of the window it is safe to put something the user has to see
or touch. On a desktop that is the whole window and asking is a formality; on a phone it is the
point.

**A modern Android app draws edge to edge whether it asks to or not.** From API 35 that is the
default and there is no opting out by targeting lower, so the surface runs under the status bar at
the top, under the gesture bar at the bottom, and under a camera cutout where there is one. Lay out
against `size()` and the buttons are drawn correctly, in the right colours, underneath the system's
own — and cannot be tapped. iOS has had the same shape since the notch.

```sysl
val safe = window.safe_area()

renderer.fill_rect(f32(safe.x), f32(safe.y), f32(safe.w), f32(safe.h))
```

**It is the alternative to going fullscreen, not a supplement to it.** `window.set_fullscreen(true)`
hides the bars and hands the program the glass, which is what a game wants. A program that keeps them
wants this instead. Fullscreen makes the safe area the whole window, so code written against it stays
correct either way — which is the argument for writing against it by default rather than only on the
platforms that need it.

Where SDL cannot answer, this returns the whole window rather than reporting a failure. That is the
one place it differs from the C, and it is deliberate: it is the only thing a caller could usefully
do with `false`, and it is what SDL itself answers on a platform with no insets.

## The display scale, which is what an interface is laid out in

`window.display_scale()` answers the factor to multiply a size by so that it comes out the same
*physical* size on every display. Write a button as 44 units tall, multiply by this, and it is a
44-unit button on a laptop panel, on an external monitor and on a phone. Write it as 44 **pixels**
and it is a thumb-sized button on one of them and a scratch on another.

```sysl
val scale = window.display_scale()
val row   = 44.0 * scale
```

**It changes while the program runs.** The window is dragged to a monitor with a different setting,
or the setting itself is changed, and `EventKind.WindowDisplayScaleChanged` says so — ask again
there rather than reading it once at startup. `EventKind.WindowDisplayChanged` is its neighbour, for
the window moving to another display at all, and carries the new display's id in `window_data1()`.

**`window.pixel_density()` is the other one and is not a substitute.** The density is only the ratio
between `size()` and `size_in_pixels()` — how many pixels back a unit of window. The scale combines
that with what the user asked for in the display's own settings, which is the part that makes a
phone's 2.75 different from a retina laptop's 2.0. Size a *texture* with the density; lay out with
the scale.

Both answer `1.0` where SDL cannot tell, rather than the `0.0` the C returns. There is nothing a
caller could do with zero except substitute one, and a zero reaching a layout multiplies every size
in it to nothing — an interface drawn perfectly at no size at all, which is a much worse failure
than an unscaled one. `display_content_scale(id)` is the same number for a display, for a program
that has to size something before it has a window to ask.

## Touch, and the tap you would otherwise handle twice

`EventKind.FingerDown`, `FingerUp`, `FingerMotion` and `FingerCanceled` are a finger on a
touchscreen or a trackpad, with `finger_x()`, `finger_y()`, the movement since the last motion, the
pressure, and the two ids that say which finger on which device.

**Most programs should ignore all four.** SDL synthesizes mouse events from touch, so a tap arrives
as `MouseButtonDown` with a position already in window coordinates — that is the whole of the input
for anything that only wants taps, it is what `sysl-lang/androidkit` does, and the same source then
runs on a desktop. These are for what a mouse cannot say: which finger, how many at once, and how
hard.

**Read both and every tap is handled twice**, which is the trap this section exists for. SDL
delivers each touch as a finger event *and* as a synthesized mouse event, so one of the pair has to
go. Each carries a sentinel naming the device it really came from:

```sysl
// Keep the finger events, drop the mouse events that were really fingers.
if e.kind() == EventKind.MouseButtonDown && e.mouse_which() == c.TOUCH_MOUSEID
    continue
```

`c.MOUSE_TOUCHID` is the mirror image — the `touch_id()` of a finger event that was really the
mouse — for a program that would rather keep the mouse events.

**A finger's position is normalized to `0..1`, and every other event's is not.** This is the one
place the two disagree about units and the disagreement is silent: `0.5` used as a coordinate lands
in the corner rather than the middle. `finger_pos_in(w, h)` does the multiplication.

```sysl
val (x, y) = e.finger_pos_in(f32(win_w), f32(win_h))
```

**`window_id()` is wrong on a touch event — use `finger_window_id()`.** SDL puts the window at the
*end* of the touch struct rather than at the front where every other variant keeps it, so the
general accessor reads the low half of the 64-bit touch id and answers a plausible number that is
not a window. It is the only place in this union where one field is not in one place, and there is
no way for a single accessor to serve both.

**`FingerCanceled` is the one most easily left out.** The system took the gesture away — an incoming
call, a notification pulled down, a system back-swipe that started at the edge — and the finger that
went down will never come up. A widget tracking a drag has to let go on it, or it stays captured by
a finger that is gone.

## The application lifecycle, which is watched rather than polled

`Terminating`, `LowMemory`, `WillEnterBackground`, `DidEnterBackground`, `WillEnterForeground` and
`DidEnterForeground` are what the system says when it, rather than the user, decides what the
program is doing. Android delivers them from `onPause`, `onResume`, `onTrimMemory` and `onDestroy`;
iOS from the matching `UIApplicationDelegate` methods. On a desktop they barely fire.

**SDL's header says each of these "must be handled in a callback set with SDL_AddEventWatch()", and
that is not advice.** They arrive synchronously from inside the platform's own callback, while the
program still has the CPU — and it may not be given it back. A `DidEnterBackground` seen through
`poll_event` is seen after the system already stopped scheduling the loop that polls, which on
Android can be after the process has been killed.

```sysl
add_event_watch(e -> if e.kind() == EventKind.DidEnterBackground then save_everything(app))
```

They reach the queue as well, which is enough for anything that only wants to stop animating.

## A text field on a phone

Three things separate a text field that works on a phone from one that only works on a desktop, and
none of them is the typing.

**Tell the platform where the field is, or the keyboard covers it.** `window.set_text_input_area(r,
cursor)` is how the system knows what to scroll above the on-screen keyboard, and where to put an
input method's suggestion window so it sits beside the text rather than over it. Without it a field
in the lower half of a phone screen disappears behind the keyboard the moment it is tapped and the
user types blind.

```sysl
window.start_text_input()
window.set_text_input_area(rect(x, y, w, h), caret_x - x)
```

Call it again whenever the caret or the layout moves. `clear_text_input_area()` when the field stops
being edited.

**Say what the field is for, or a PIN field offers a full QWERTY.**

```sysl
window.start_text_input_with(TextInputType.Email)
window.start_text_input_with(TextInputType.PinHidden)
```

Nine types — text, name, e-mail, username, password hidden or visible, number, PIN hidden or
visible. On a desktop none of it changes anything, which is exactly why it is easy to leave out and
be told about by somebody holding a phone.

The three optional properties are `Option` because **SDL derives them from the type when they are not
set**, and passing a value where you meant "let SDL decide" is worse than not passing one:
capitalization defaults to sentences for ordinary text, words for a name, and *none* for an e-mail
address, a username or a password. Send `Capitalization.Sentences` to every field and you capitalize
e-mail addresses.

```sysl
window.start_text_input_with(TextInputType.Text, Some(Capitalization.Words), Some(false), Some(true))
```

**Know whether the keyboard is up.** `window.screen_keyboard_shown()` for right now, and
`has_screen_keyboard_support()` for whether this machine has one at all — the second is what decides
whether a layout needs to reserve space for it, and a desktop answers false to both.

**And for an input method, the composing region.** `EventKind.TextEditing` carries the text being
formed before it commits; `composition_start()` and `composition_length()` are the part of it the
input method has selected, which is what draws the underline a CJK keyboard shows mid-word. Both are
`-1` where the platform does not say, which is ordinary — underline the whole composition then.
`window.clear_composition()` throws away a composition in progress, which a field does when it loses
focus so the half-formed text does not commit into whatever takes focus next.

`window.text_input_active()` answers whether SDL thinks the field is being edited, so a widget need
not keep that flag itself and keep it synchronised.

## Light or dark

```sysl
val bg = if system_theme() == SystemTheme.Dark then rgb(18, 18, 22) else rgb(250, 250, 252)
```

`Unknown` is an ordinary answer rather than a failure — a platform with no such setting reports it,
so an interface wants a scheme it falls back to rather than an error path.

**Ask again on `EventKind.SystemThemeChanged`.** A phone switches at sunset on a schedule the program
is never told in advance, and a desktop switches when the user does. Read once at startup and the
interface is the wrong colour for the rest of the session. `EventKind.LocaleChanged` is its
neighbour, for `preferred_locales()`.

## Opening a URL

`open_url("https://sysl.sh")` hands a URL to whatever the system opens it with — a browser for
`https:`, a mail client for `mailto:`, the file manager for `file:`. It is how a link in an interface
works at all, since a program drawing its own pixels has no link for the platform to notice.

**`true` means the system accepted it, not that the user saw anything**, and nothing comes back. On a
phone the program is backgrounded by the act of opening one, so the lifecycle events fire.

## Text with nothing installed

`renderer.debug_text(x, y, "hello")` draws a line in a fixed 8x8 bitmap font **carried inside SDL
itself**, so a program that only wants to say something needs no font file, no `sdl3-ttf`, and
nothing shipped beside the binary.

```sysl
renderer.set_draw_color(rgb(220, 220, 230))
renderer.debug_text(16.0, 16.0, "hello from sysl")
```

It draws in the current draw colour, so `set_draw_color` is what chooses the ink, and
`c.DEBUG_TEXT_FONT_CHARACTER_SIZE` is 8 for a caller that wants to compute a width.

**SDL's own documentation is blunt that it is for debugging rather than for an interface**: one size,
one face, ASCII, no shaping, no wrapping. Where any of that matters `sdl3-ttf` is the answer and this
is not. What it is very good at is the first thing a new program does — putting a legible line on the
screen before anything else works.

## Reading the frame back

`renderer.read_pixels()` answers what is in the target as a `&Surface`, which is what a screenshot,
a colour picker and a test that checks *where* a call drew are all written with. `read_pixels_rect`
takes one rectangle instead, in the target's own pixels.

**Read before `present`, not after** — presenting is allowed to leave the backbuffer undefined, so a
shot taken afterwards is a bet on the driver. And it is slow by nature: it waits for the GPU and
pulls the pixels back across the bus, so it belongs on a key rather than in a frame loop.

```sysl
val shot = renderer.read_pixels().expect("the frame")

save_png(shot, "screenshot.png")        // sdl3-image
```

## Events

`SDL_Event` is a union of about thirty structs in 128 bytes. `c` declares it as the header every
event begins with plus a view per variant, and the accessors here reinterpret into whichever view a
field belongs to — so there is no shim and no hand-written offset arithmetic at a call site.

A field is only meaningful for the matching kind. That is what a union means and no binding can make
it otherwise: match on `kind()` first.

## Audio

Playback is on the **queue model**: SDL runs its own audio thread and pulls from the stream, and the
program pushes a finished buffer. Nothing in this package is ever called from SDL's audio thread.

That is deliberate. SDL's other model hands the device a callback it invokes on a thread sysl did
not start, which would mean adjusting reference counts outside anything this package has established
the atomicity of. The queue model asks that of nobody and costs a buffer's latency. `sdl3-mixer` is
the answer for anything more.

## Tests

```
sysl test .
```

Fifty-five tests, run headless against a real SDL3 — the dummy video and audio drivers create
windows, renderers, textures and devices and draw into memory, so nothing here needs a display or a
sound card.

**Half of them exist to pin the enumerations.** Every `code` and `of` is a `match` against a name
the C compiler resolved, so a *typo* is impossible — but that the right variant sits on the right
side of an arrow is checked by nothing at all, and a mapping with two lines swapped compiles, links
and asks SDL for the wrong thing. So each enumeration is walked both ways, and several are pushed
through SDL itself: the name it gives a format, the flags it reports for a window it made, the
number it stores in a texture's property bag, and the pixel that comes back after a blend — which is
the only way to see that `BlendMode.Blend` reached the library rather than merely type-checked.

The struct layouts are pinned the same way: `SDL_Event` is 128 bytes, `SDL_Surface` 48, `SDL_Vertex`
32, and every variant view is built as C lays it out and read back through the public accessors, so
a field at the wrong offset fails a test rather than answering plausible nonsense.

## License

ISC — see [LICENSE](LICENSE).
