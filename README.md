# Skitter

**A sysl application on a phone, and the parts of that which are not the application's.**

```
dependencies {
  skitter { git = "github.com/sysl-lang/skitter", version = "0.3.0" }
}
```

Naming this one reaches SDL3 through it, because imports are transitive. It needs **sysl 0.0.161**.

**If you are starting a project, you do not start here.** Clone
[`sysl-lang/skitter-app`](https://github.com/sysl-lang/skitter-app), change two lines, and write your
program — it names this package for you. This repository is the framework that makes that possible,
and it is worth reading only if you want to know how.

## What is here, and what is not

SDL3 already covers nearly all of a phone: a tap arrives as a mouse event, typed text arrives
composed by the platform, the renderer and the texture are the same calls a desktop makes. The
binding in `sysl-lang/sdl3` exposes the soft keyboard pleasantly too — `start_text_input`,
`stop_text_input`, `screen_keyboard_shown` — so **Skitter does not wrap any of it**. A framework that
re-exported what was already good would be a layer to look through rather than one to use.

Four things are left: three because an application got them wrong first, and the requests a program
makes of the phone itself, which reach Java or nothing.

### The system bars

**From Android API 35 an application draws edge to edge whether it asks to or not.** The window runs
under the status bar and the navigation bar, so a program laying itself out against the window's own
size draws its first and last rows underneath both — correctly, and invisibly.

**`Window.safe_area` is the obvious answer and it is the wrong rectangle for a drawing.** SDL builds
Android's safe area out of five inset types at once — `systemBars`, `systemGestures`,
`mandatorySystemGestures`, `tappableElement` and `displayCutout` — because the question it answers is
*where can a button go*. Measured on a gesture-navigation phone that is **78 pixels off each side**
for the back-gesture strips, and a bottom inset reaching well above the navigation bar. For something
being looked at rather than touched, all of it is too conservative.

SDL exposes only the combined rectangle and no way to ask for one of the five, so the bars reach a
program through JNI or they do not reach it at all.

```
import sh.sysl.skitter.safe_area

val safe = safe_area(renderer)      // in pixels, asked every frame
```

### Why that makes Skitter a package rather than advice

**JNI binds a native method by mangling the class's package and name into the symbol it looks up.**
So every application that wrote its own bridge wrote a *different* symbol, by hand, kept in step with
a Scala file it also had to rename — and getting it wrong links cleanly and dies at the first call
with an `UnsatisfiedLinkError`, because JNI resolves at run time.

**Skitter fixes the class at `sh.sysl.skitter.SkitterActivity`, which fixes the symbol.** A launcher
activity does not have to be a class in the application's own package — `android:name` takes any
class on the classpath — so the bridge can live in a library, and an application's `applicationId`
becomes a free string that nothing else has to agree with.

Nothing has to force the symbol into the link, either: every sysl module across every dependency
compiles into **one object** in the archive, so the `-u SDL_main` an Android build already passes
pulls in the member the bridge is in.

### The orientation pair

**One setting spread across two calls, and neither announces the other.** Naming every orientation
looks sufficient and is not: `SDLActivity.setOrientationBis` only promotes "all four allowed" into
*follow the device* when the window is **resizable**. For a fixed-size window it goes on choosing
from the requested width and height, so a program that asks for 800 by 600 and names all four still
comes up locked in landscape.

```
orient()                                        // before init
create_window(title, w, h, window_flags())      // folds in WINDOW_RESIZABLE
```

Both are harmless off Android — a desktop SDL has no such hint and ignores it — so a program that
runs in both places calls them unconditionally.

### Asking the phone for things

```
keep_awake(on: bool) -> bool                                  // the screen stays on, or may sleep
vibrate(ms: int) -> bool                                      // a buzz
tick() -> bool                                                // the light tick a picker gives
open_url(url: string) -> bool                                 // a browser, a mail client
share_text(text: string) -> bool                              // Android's share sheet
permission_granted(name: string) -> bool                      // asks, never prompts
request_permission(name: string, answer: &sync Fn(bool) -> unit) -> bool   // prompts
```

**None of it is JNI on the sysl side.** `keep_awake` is SDL's screensaver switch, which on Android
is `FLAG_KEEP_SCREEN_ON`; `open_url` is `SDL_OpenURL`; `request_permission` is
`SDL_RequestAndroidPermission`. The rest are numbers posted with `SDL_SendAndroidMessage`, which SDL
delivers to `SkitterActivity.onUnhandledMessage` on the UI thread — a string rides beside the number
as an SDL hint the activity reads back, and the one answer that has to come back, whether a
permission is granted, returns through an exported method the way the bars do.

**A permission's name is `skitter.permissions`' word** — `microphone`, `camera`, `internet`,
`bluetooth`, `vibrate` — or Android's own spelling with a dot in it. `request_permission`'s answer
may arrive on another thread, which is why it is a `&sync Fn`: it says what it learned by writing to
something the program's loop reads.

**Whether the screen starts awake is `skitter.keepAwake` in `gradle.properties`**, `false` by
default; `keep_awake` changes it while the program runs.

**Everything here is harmless on a desktop**, so a program that also runs there calls it
unconditionally: the screen and a URL are SDL's everywhere, a buzz and a share answer `false`, every
permission is granted, and a request is answered `true` at once.

## What an application still writes

**The entry point, and that is all.** Android has no `main`: `SDLActivity` loads `libmain.so` and
looks up `SDL_main`, so what an Android program needs is an entry point the Java half can *find*
rather than one that runs first.

```
@export("SDL_main")
sdl_main(argc: i32, argv: **u8) -> i32 = ...
```

That is also why an application built this way carries **no C at all**: every other SDL project
carries a shim defining `SDL_main` in C, and sysl defines the symbol directly.

## Using it with syslUI

`sysl-lang/syslui-sdl` owns its own window and frame loop and keeps the insets it lays out against in
`set_insets`. Skitter receives them from Android; forwarding is one line, in the frame or beside it:

```
set_insets(ins.left, ins.top, ins.right, ins.bottom)
```

**A callback would be nicer and is not available**: `sysl build-c` refuses module storage that an
initializer would have to fill, because a C project linking the archive supplies its own `main` and
nothing would ever run it — so Skitter holds four bare `int`s and is *pulled* rather than pushing.
Filed as card `0263`.

## The insets are pulled, not pushed

```
insets() -> Insets            // left, top, right, bottom, in pixels
safe_area(renderer) -> FRect  // the drawable minus the bars
fit(w, h, ins) -> FRect       // the same arithmetic, without a renderer
```

All zero until Android reports them, which it does before the first frame and again on every
rotation — and all zero is also the honest answer under `sysl run` on a desktop, where the program
has a whole window.

**`fit` clamps and never returns a negative rectangle.** Some devices report the new bars before SDL
reports the new window, so for a frame or two after a rotation the insets exceed the size they are
applied to — and the renderer does not clip a negative `FRect`, it draws it inside out.

## Settings that survive a restart

```
val prefs = app_prefs("sh.sysl", "tuner")
val a4 = signal(prefs.real("a4", 440.0))     // a default when nothing is stored

a4.set(v)                                     // wherever the slider writes it
prefs.set_real("a4", v)
```

`real`, `int`, `bool` and `string` read with a default; `set_real`, `set_int`, `set_bool` and
`set_string` write. **The file is `prefs.toml` in SDL's preference directory** — the app's private
internal storage on Android, `~/Library/Application Support/<org>/<app>/` on macOS — and is one flat
TOML table a person can open and edit. `prefs_in(dir)` keeps it anywhere else.

**Every set writes the file, atomically, unless the value did not change.** Android stops a
backgrounded app without asking, so a store waiting for a `save()` at exit would lose what changed
since; the write goes to a pending name and is renamed, so a kill mid-write leaves the old file
whole. **Nothing here crashes**: a missing or corrupt file, or a value of the wrong type, answers the
default, and a write that fails is kept in `error()` while the value is still held.

## Logging that reaches logcat

```
logcat("musicbox")          // first thing in SDL_main
```

**On a phone, standard output and standard error go nowhere** — Android points both at `/dev/null`
— so `logcat(tag)` installs a `sysl.log` sink over `__android_log_write`. Every `sysl.log` record,
from the program or from any library under it, becomes one logcat entry under `tag`: Debug, Info,
Warn and Error at Android's priorities 3 to 6, the message and its fields as the text, and no time
or level of its own, logcat stamping both. `adb logcat -s musicbox` reads them back. **On a desktop
the line does nothing**, and records go on reaching standard error.

**The application's CMake has to link `log`**:

```
target_link_libraries(main ... log)
```

The sink's module says `@link("log")`, but Gradle's CMake owns the final link and never sees a sysl
link directive.

**`print` is not redirected, deliberately** — `logcat.sysl`'s header says why: standard output is
fully buffered once it is not a terminal, and changing that is legal only before the first write,
which a library cannot know has not happened.

## Tests

```
sysl test .
```

Eighteen, over the part that can be wrong without anything crashing: the arithmetic that turns bars
into a rectangle, the storage the bridge writes, that `window_flags` cannot lose `WINDOW_RESIZABLE`,
the permission names and the command numbers the activity matches on, the answer that comes back
from it, and what each request does on a desktop. Thirteen more over the settings store, each
against a temporary directory of its own, and six over what the logcat sink hands `liblog`: the
priorities, a tag terminated where it ends, and the line it renders.

**Whether JNI finds the symbol is not testable here and is not pretended to be** — it is decided by a
class name in an APK on a device, and a test that mocked the lookup would be asserting the mock. That
half is proved by `sysl-lang/skitter-app` running on hardware.

## Licence

ISC. SDL3 is zlib and is not carried here.
