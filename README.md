# nautilus-scripts
Some `nautilus` (GNOME file manager) scripts i made for myself but they should work for most linux setups!

# Usage
```bash

# Create scripts folder
mkdir ~/.local/share/nautilus/scripts/

# Move the script
mv <SCRIPT_NAME> ~/.local/share/nautilus/scripts/

# Make it executable
chmod +x ~/.local/share/nautilus/scripts/<SCRIPT_NAME

# Create keybind file
touch ~/.config/nautilus/scr-keybinds

# Set keybind
echo "<KEYBIND> <SCRIPT_NAME>" >> ~/.local/share/nautilus/scripts/<SCRIPT_NAME
```
# Scripts
- [Terminal at Folder Location](#open-terminal)
  
- - -

- [Fast Video Preview](#ffpreview)
- [Fast Text Preview](#txtpreview)
- [Fast Image Preview](#imvpreview)

**DISCLAIMER: Preview scripts works better on a _Niri_ setup, since they autoclose after losing focus. You need the [niri](niri/) version of the scripts though. Also these window rules should make the experience much better:**

<details>
<summary>Window rules</summary>
    
```txt
window-rule {
    match app-id="imv"
    open-floating true
    default-column-width { proportion 0.4; }
    default-window-height { proportion 0.6; }
}

window-rule {
    match app-id="text-preview"
    open-floating true
    default-column-width { proportion 0.4; }
    default-window-height { proportion 0.6; }
}

window-rule {
    match app-id="ffplay"
    open-floating true
    default-column-width { proportion 0.5; }
    default-window-height { proportion 0.5; }
}
```

</details>

- - -
- [OCR From Selected Image](#ocr)


## open-terminal

Open terminal at the current folder location.

```bash
#!/usr/bin/env bash
TARGET_DIR="$PWD"
SELECTED="${1:-${NAUTILUS_SCRIPT_SELECTED_FILE_PATHS:-}}"
SELECTED=$(echo -n "$SELECTED" | head -n 1)

if [ -n "$SELECTED" ] && [ -d "$SELECTED" ]; then
    TARGET_DIR="$SELECTED"
fi

TERMINAL=${TERMINAL:-x-terminal-emulator}
if ! command -v "$TERMINAL" >/dev/null 2>&1; then
    for term in alacritty kitty gnome-terminal konsole xfce4-terminal; do
        if command -v "$term" >/dev/null 2>&1; then
            TERMINAL="$term"
            break
        fi
    done
fi

cd "$TARGET_DIR" || exit 1
"$TERMINAL" &
```

## ffpreview

Video preview using `ffplay` from `ffmpeg`. **Requires `ffmpeg` to be installed**.

```bash
#!/usr/bin/env bash
FILE="${1:-${NAUTILUS_SCRIPT_SELECTED_FILE_PATHS:-}}"
FILE=$(echo -n "$FILE" | head -n 1)
[ -n "$FILE" ] || exit 0

ffplay -v quiet -autoexit -window_title "ffplay-preview" -x 1280 -y 720 "$FILE" &
```
![ffplay](https://github.com/user-attachments/assets/20d96607-326d-4c5d-af0c-594a3ead3fe2)

## txtpreview

Text file preview using `bat`. **Requires `bat` to be installed**.

![bat](https://github.com/user-attachments/assets/0108610d-148d-401b-aeaf-9343697d665d)

## imvpreview

Image file preview using `imv`. **Requires `imv` to be installed**.

## ocr

Copy text from selected image file using `tesseract`. **Requires `tesseract`, `tesseract-data-eng`(or other languages) and `imagemagick` to be installed**.

```bash
#!/usr/bin/env bash
FILE="${1:-${NAUTILUS_SCRIPT_SELECTED_FILE_PATHS:-}}"
FILE=$(echo -n "$FILE" | head -n 1)
[ -n "$FILE" ] || exit 0

TERMINAL=${TERMINAL:-x-terminal-emulator}
if ! command -v "$TERMINAL" >/dev/null 2>&1; then
    for term in alacritty kitty gnome-terminal konsole xfce4-terminal; do
        if command -v "$term" >/dev/null 2>&1; then
            TERMINAL="$term"
            break
        fi
    done
fi

"$TERMINAL" -e bash -c 'bat --color=always --style=numbers --paging=auto "$0"; read -rsn1' "$FILE" &
```

# Setup Specialized Versions
## Niri
I am currently using a Niri setup, these are the scripts specialized for the specific setup. They should work on your Niri setup too out-of-the-box or with some small adjustments.
Refer to the: [niri](/niri/)
### Differences
- Their window automatically close with this `niri` config:
```txt
window-rule {
    match app-id="text-preview"
    open-floating true
    default-column-width { proportion 0.4; }
    default-window-height { proportion 0.6; }
}

window-rule {
    match app-id="ffplay"
    open-floating true
    default-column-width { proportion 0.5; }
    default-window-height { proportion 0.5; }
}
```
