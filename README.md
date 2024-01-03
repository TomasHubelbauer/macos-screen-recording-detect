# Detect Screen Recording in macOS

macOS has a built-in screen recording feature (Cmd+Shift+5).
When the screen recording ends, a new file is created in the Desktop directory.
While the screen recording is ongoing, a temporary file exists in the
`/Users/$USER/Library/ScreenRecordings/` (`~/Library`) directory.

This path was discovered using `lsof | grep '\.mov$'` while a screen recording
was ongoing:

```
$USER@M1-MBP ~ % lsof | grep '\.mov$'
screencap $IPD $USER $FD$MODE REG $DEVICE $SIZE $NODE ~/Library/ScreenRecordings/$NAME.mov
```

This also tells us that the process name of the screen recording utility is
`screencap`.

To detect whether a screen recording initiated via the screen recording feature
of macOS is in progress, we can list open files of `screencan` that end in the
MOV extension.

```
lsof -c screencap | grep '\.mov$'
```

To learn more about the `screencap` process, specifically its executable path,
we can use `ps -p $PID`.
This reveals the executable is at `/usr/sbin/screencapture`.

This is the same executable as the $PATH-included `screencapture` which can be
used for screenshots but using it for screen recording is not officially
documented as far as I can tell.

We can see what CLI options this executable accepts using `man screencapture`
or `screencapture --help`.

The `ps -p $PID` line also tells us the command line arguments the process was
started with: `-pdiU -z keyboard.interactive`.

Let's break down what this does using `--help`:

- `-p`:
  Screen capture will use the default settings for capture.
  The files argument will be ignored.
- `-d`: Display errors to the user graphically.
- `-i`: Capture screen interactively, by selection or window.
- `-U`: Show interactive toolbar in interactive mode.
- `-z`: This option is not documented in `screencapture --help`.

Checking the man page I don't see a section for it either.

I did however notice there is a `-v` toggle which leads me to believe it
might be possible to start a screen recording programatically.
Only taking screenshots using the `screencapture` CLI is documented online.
I have not found a way to start a screen recording using the CLI yet.

## Extras

### Associating the temporary file with the final screen recording file

The temporary recording file's name is a UUID.
It doesn't have the date and time in the name so we can't use it to scan
for the final file after the recording has finished.

To find which final file corresponds to the temporary recording file, we
can use the `inode`.

`stat -f %i "$HOME/Desktop/Screen Recording ….mov"`

The `inode` number given by this will be the same as the `inode` number
of the final recording, indicating the file is moved after the recording
finishes.

### `screencapture` PLIST

There is also a binary PLIST used by `screencapture`.
It can be viewed in a human-readable form like so:

```
plutil -convert xml1 -o - ~/Library/Preferences/com.apple.screencapture.plist
```

I haven't found it to contain super interesting information, however.

## To-Do

### Figure out how to start screen recording using the CLI

I tried `screencapture -v file.mov` and got this error:

```
type any character (or ctrl-c) to stop screen recording
screencapture: capture error The operation could not be completed
```

Maybe this could solve it? https://apple.stackexchange.com/a/465441/193285
