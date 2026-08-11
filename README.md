# sdl3

SDL3 for sysl — a window, an accelerated renderer, the event queue, keyboard and mouse, the
clipboard, the system file dialog, and queued audio.

```
dependencies {
  sdl3 { git = "github.com/sysl-lang/sdl3", version = "0.1.0" }
}
```

```sysl
import sh.sysl.sdl3.*

main()
    init(INIT_VIDEO)

    val window = create_window("hello", 640, 480, WINDOW_RESIZABLE).expect("a window")
    val renderer = window.create_renderer().expect("a renderer")

    var running = true

    while running
        while true
            poll_event() match
                Some(e) -> if e.kind == EVENT_QUIT then running = false
                None -> break

        renderer.clear_to(rgb(20, 20, 30))
        renderer.set_draw_color(rgb(220, 80, 60))
        renderer.fill_circle(320.0, 240.0, 80.0, rgb(220, 80, 60))
        renderer.present()

    renderer.destroy()
    window.destroy()
    quit()
```

## Installing SDL3, and the two flags

Nothing is vendored here. SDL is a large C project with its own build system and its own platform
backends, so a package carrying it would be maintaining a port rather than binding a library.

```
brew install sdl3                       # or the distribution's package
```

A build then needs the prefix on the command line, because a toolchain searches its own directories
and Homebrew's is not one of them:

```
sysl run prog.sysl --include-path /opt/homebrew/include --link-path /opt/homebrew/lib
```

That is deliberate rather than a gap. `design/15 §8` refuses both a `@link_path` attribute and a
`package.hocon` field for it: where a prefix lives is a fact about somebody's laptop, not a property
of the package. `LIBRARY_PATH` and `CPATH` work too, since clang reads them, and are the better
answer on a machine where the setting never changes.

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

## What it looks like

**A handle is a one-field struct over the pointer SDL gave.** `Window`, `Renderer`, `Texture`,
`Surface`, `Cursor` and `AudioStream` cost nothing at run time and carry their methods.

**Anything that creates one answers `Option`.** SDL says "failed" with a null pointer and the reason
in `error()`; an `Option` is the same information in the shape the language already reads, and a
null handle cannot be passed on by accident.

**Everything else answers `bool`, exactly as SDL does.** A binding that invented a richer failure
type than the library gives would be inventing the failure cases too. When something answers
`false`, `error()` says why.

**Nothing has a destructor; `destroy` is written where the program decides the thing is finished.**
A destructor in sysl runs for a value behind a `&T`, so it would mean a heap box per texture — a
real cost in a draw loop, for a resource whose lifetime the program already knows.

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

add_event_watch(e -> if e.kind == EVENT_QUIT then t.n += 1)
```

## Events

`SDL_Event` is a union of about thirty structs in 128 bytes. It is read here as one struct holding
the header every event begins with, plus a view per variant that the accessors reinterpret into — so
there is no shim and no hand-written offset arithmetic at a call site.

A field is only meaningful for the matching `kind`. That is what a union means and no binding can
make it otherwise: match on `kind` first.

```sysl
poll_event() match
    Some(e) ->
        if e.kind == EVENT_KEY_DOWN && e.key_scancode() == SCANCODE_ESCAPE then running = false
        if e.kind == EVENT_MOUSE_BUTTON_DOWN then click(e.mouse_x(), e.mouse_y())
    None -> ()
```

## Audio

Playback is on the **queue model**: SDL runs its own audio thread and pulls from the stream, and the
program pushes a finished buffer. Nothing in this package is ever called from SDL's audio thread.

That is deliberate. SDL's other model hands the device a callback it invokes on a thread sysl did
not start, which would mean adjusting reference counts outside anything this package has established
the atomicity of. The queue model asks that of nobody and costs a buffer's latency. `sdl3-mixer` is
the answer for anything more.

## Tests

```
sysl test . --include-path /opt/homebrew/include --link-path /opt/homebrew/lib
```

Thirty tests, run headless against a real SDL3 — the dummy video and audio drivers create windows,
renderers, textures and devices and draw into memory, so nothing here needs a display or a sound
card.

**Half of them exist to pin the transcribed constants.** Every `SDL_INIT_*`, window flag, pixel
format, colorspace, scancode and event type is a `#define` with no symbol, so it was copied by hand
— and a value that is wrong is usually wrong *quietly*: a pixel format that decodes to the wrong
colours, a colorspace that shifts greens, an event type that simply never matches. Each is checked
against something SDL itself computes from its own header — the name it gives a format, the flags it
reports for a window it made, the number it stores in a texture's property bag — so nothing is
trusted because it was typed carefully.

The struct layouts are pinned the same way: `SDL_Event` is 128 bytes, `SDL_Surface` 48, `SDL_Vertex`
32, and every variant view is built as C lays it out and read back through the public accessors, so
a field at the wrong offset fails a test rather than answering plausible nonsense.

## License

ISC — see [LICENSE](LICENSE).
