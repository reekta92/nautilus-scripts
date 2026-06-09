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
- [Universal File Preview](#preview)
- [Fast Video Preview](#ffpreview)
- [Fast Text Preview](#txtpreview)
- [Fast Image Preview](#imvpreview)

Preview scripts auto-close when they lose window focus on Niri. The `niri/` folder contains identical copies for convenience. Also these window rules should make the experience much better:

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

window-rule {
    match app-id="dev.ghostty.preview"
    open-floating true
    default-column-width { proportion 0.4; }
    default-window-height { proportion 0.6; }
}
```

</details>

- [OCR From Selected Image](#ocr)

## open-terminal

Open terminal at the current folder location.

```bash
#!/usr/bin/env bash

detect_terminal() {
    if [ -n "$TERMINAL" ] && command -v "$TERMINAL" >/dev/null 2>&1; then
        echo "$TERMINAL"
        return 0
    fi
    for t in ghostty kitty alacritty wezterm foot konsole xfce4-terminal gnome-terminal; do
        if command -v "$t" >/dev/null 2>&1; then
            echo "$t"
            return 0
        fi
    done
    return 1
}

TARGET_DIR="$PWD"
SELECTED=$(echo -n "$NAUTILUS_SCRIPT_SELECTED_FILE_PATHS" | head -n 1)

if [ -n "$SELECTED" ]; then
    if [ -d "$SELECTED" ]; then
        TARGET_DIR="$SELECTED"
    else
        TARGET_DIR="$(dirname "$SELECTED")"
    fi
fi

TERM=$(detect_terminal) || exit 1

case "$TERM" in
    kitty) kitty --directory="$TARGET_DIR" & ;;
    wezterm) wezterm start --cwd "$TARGET_DIR" & ;;
    konsole) konsole --workdir "$TARGET_DIR" & ;;
    *) "$TERM" --working-directory="$TARGET_DIR" & ;;
esac
```

![terminal](https://github.com/user-attachments/assets/520c263a-7773-4507-8d44-ec6ec70f0bf4)

## preview

Universal file preview that detects MIME type and opens the appropriate viewer. Supports images (imv), video/audio (ffplay), PDF, DOCX, DOC, ODT, archives (zip/tar), directories (eza), markdown (glow), and generic text (bat). Opens in your terminal with auto-close on unfocus via niri.

**Requires `bat`, `jq`, `niri` to be installed** (plus optional: `imv`, `ffmpeg`, `pdftotext`, `docx2txt`, `catdoc`, `odt2txt`, `glow`, `eza`).

```bash
#!/usr/bin/env bash

detect_terminal() {
    if [ -n "$TERMINAL" ] && command -v "$TERMINAL" >/dev/null 2>&1; then
        echo "$TERMINAL"
        return 0
    fi
    for t in ghostty kitty alacritty wezterm foot konsole xfce4-terminal gnome-terminal; do
        if command -v "$t" >/dev/null 2>&1; then
            echo "$t"
            return 0
        fi
    done
    return 1
}

TERM=$(detect_terminal) || exit 1

# Get the first selected file
FILE=$(echo -n "$NAUTILUS_SCRIPT_SELECTED_FILE_PATHS" | head -n 1)
[ -n "$FILE" ] || exit 0;

# Helper: Focus tracking
auto_close_on_unfocus() {
    local PID=$1

    # Phase 1: Wait for window to exist and be focused
    local count=0
    while kill -0 $PID 2>/dev/null; do
        FOCUSED_ID=$(niri msg -j focused-window | jq '.id' 2>/dev/null)
        WINDOW_ID=$(niri msg -j windows | jq ".[] | select(.pid == $PID) | .id" 2>/dev/null)

        if [ -n "$WINDOW_ID" ] && [ "$FOCUSED_ID" == "$WINDOW_ID" ]; then
            break
        fi

        count=$((count + 1))
        if [ $count -gt 50 ]; then
            break
        fi
        sleep 0.1
    done

    # Phase 2: Kill when it loses focus
    while kill -0 $PID 2>/dev/null; do
        FOCUSED_ID=$(niri msg -j focused-window | jq '.id' 2>/dev/null)
        WINDOW_ID=$(niri msg -j windows | jq ".[] | select(.pid == $PID) | .id" 2>/dev/null)

        if [ -n "$WINDOW_ID" ] && [ "$FOCUSED_ID" != "$WINDOW_ID" ]; then
            kill $PID 2>/dev/null
            break
        fi
        sleep 0.2
    done
}

# Helper: Launch a command in terminal, set window title, keep open until keypress.
preview_in_terminal() {
    local TITLE=$1
    local CMD=$2

    case "$TERM" in
        ghostty) "$TERM" --class="dev.ghostty.preview" --gtk-single-instance=false -e bash -c "printf '\033]0;%s\007' \"$TITLE\"; $CMD; printf '\n[Press any key to close]'; read -rsn1" "$FILE" & ;;
        kitty) "$TERM" --class "file-preview" --title "$TITLE" -e bash -c "printf '\033]0;%s\007' \"$TITLE\"; $CMD; printf '\n[Press any key to close]'; read -rsn1" "$FILE" & ;;
        *) "$TERM" -e bash -c "$CMD; printf '\n[Press any key to close]'; read -rsn1" "$FILE" & ;;
    esac
    auto_close_on_unfocus $! &
}

# Detect MIME type
MIME=$(file --dereference --mime-type -b "$FILE")

case "$MIME" in
    image/*)
        if command -v imv >/dev/null; then
            imv "$FILE" &
            auto_close_on_unfocus $! &
        else
            notify-send "preview" "imv not found"
        fi
        ;;
    video/*)
        if command -v ffplay >/dev/null; then
            SDL_VIDEODRIVER=wayland ffplay -v quiet -autoexit -window_title "preview" -x 1280 -y 720 "$FILE" &
            auto_close_on_unfocus $! &
        else
            notify-send "preview" "ffplay not found"
        fi
        ;;
    audio/*)
        if command -v ffplay >/dev/null; then
            SDL_VIDEODRIVER=wayland ffplay -v quiet -autoexit -nodisp "$FILE" &
        else
            notify-send "preview" "ffplay not found"
        fi
        ;;
    application/pdf|application/x-pdf)
        preview_in_terminal "PDF" 'pdftotext "$0" - | bat -l txt -p --color=always --paging=auto'
        ;;
    application/vnd.openxmlformats-officedocument.wordprocessingml.document)
        preview_in_terminal "DOCX" 'docx2txt "$0" - | bat -l txt -p --color=always --paging=auto'
        ;;
    application/msword)
        preview_in_terminal "DOC" 'catdoc "$0" | bat -l txt -p --color=always --paging=auto'
        ;;
    application/vnd.oasis.opendocument.text)
        preview_in_terminal "ODT" 'odt2txt "$0" | bat -l txt -p --color=always --paging=auto'
        ;;
    application/zip)
        preview_in_terminal "Archive" 'unzip -l "$0" | bat -p --color=always --paging=auto'
        ;;
    application/x-tar|application/gzip|application/x-xz|application/x-bzip2|application/x-zstd)
        preview_in_terminal "Archive" 'bsdtar -tf "$0" | bat -p --color=always --paging=auto'
        ;;
    inode/directory)
        preview_in_terminal "Directory" 'eza --tree --level=3 --color=always "$0" | bat -p --paging=auto'
        ;;
    *)
        EXT="${FILE##*.}"
        EXT=$(echo "$EXT" | tr '[:upper:]'[:lower:]')
        if [ "$EXT" == "md" ] || [ "$MIME" == "text/markdown" ]; then
             preview_in_terminal "Markdown" 'glow "$0"'
        else
             preview_in_terminal "Text" 'bat -p --color=always --style=numbers --paging=auto "$0"'
        fi
        ;;
esac
```

## ffpreview

Video preview using `ffplay` from `ffmpeg`.

**Requires `ffmpeg` to be installed**.

```bash
#!/usr/bin/env bash
FILE=$(echo -n "$NAUTILUS_SCRIPT_SELECTED_FILE_PATHS" | head -n 1)
[ -n "$FILE" ] || exit 0

SDL_VIDEODRIVER=wayland ffplay -v quiet -autoexit -window_title "ffplay-preview" -x 1280 -y 720 "$FILE" &
FF_PID=$!

while kill -0 $FF_PID 2>/dev/null; do
    if niri msg focused-window 2>/dev/null | grep -qiE 'app_id: "ffplay"|title: "ffplay-preview"'; then
        break
    fi
    sleep 0.1
done

while kill -0 $FF_PID 2>/dev/null; do
    if ! niri msg focused-window 2>/dev/null | grep -qiE 'app_id: "ffplay"|title: "ffplay-preview"'; then
        kill $FF_PID 2>/dev/null
        break
    fi
    sleep 0.2
done
```
![ffplay](https://github.com/user-attachments/assets/20d96607-326d-4c5d-af0c-594a3ead3fe2)

## txtpreview

Text file preview using `bat`.

**Requires `bat` to be installed**.

```bash
#!/usr/bin/env bash

detect_terminal() {
    if [ -n "$TERMINAL" ] && command -v "$TERMINAL" >/dev/null 2>&1; then
        echo "$TERMINAL"
        return 0
    fi
    for t in ghostty kitty alacritty wezterm foot konsole xfce4-terminal gnome-terminal; do
        if command -v "$t" >/dev/null 2>&1; then
            echo "$t"
            return 0
        fi
    done
    return 1
}

FILE=$(echo -n "$NAUTILUS_SCRIPT_SELECTED_FILE_PATHS" | head -n 1)
[ -n "$FILE" ] || exit 0

TERM=$(detect_terminal) || exit 1

case "$TERM" in
    ghostty) "$TERM" --class="text-preview" -e bash -c 'bat -p --color=always --style=numbers --paging=auto "$0"; read -rsn1' "$FILE" & ;;
    kitty) "$TERM" --class "text-preview" --title "text-preview" -e bash -c 'bat -p --color=always --style=numbers --paging=auto "$0"; read -rsn1' "$FILE" & ;;
    *) "$TERM" -e bash -c 'bat -p --color=always --style=numbers --paging=auto "$0"; read -rsn1' "$FILE" & ;;
esac
TXT_PID=$!

while kill -0 $TXT_PID 2>/dev/null; do
    if niri msg focused-window 2>/dev/null | grep -qi 'text-preview'; then
        break
    fi
    sleep 0.1
done

while kill -0 $TXT_PID 2>/dev/null; do
    if ! niri msg focused-window 2>/dev/null | grep -qi 'text-preview'; then
        kill $TXT_PID 2>/dev/null
        break
    fi
    sleep 0.2
done
```

![bat](https://github.com/user-attachments/assets/0108610d-148d-401b-aeaf-9343697d665d)

## imvpreview

Image file preview using `imv`.

**Requires `imv` to be installed**.

```bash
#!/usr/bin/env bash
FILE="${1:-${NAUTILUS_SCRIPT_SELECTED_FILE_PATHS%%$'\n'*}}"

[ -n "$FILE" ] || exit 0

if command -v imv >/dev/null 2>&1; then
    imv "$FILE" &
    IMV_PID=$!

    (
        while kill -0 $IMV_PID 2>/dev/null; do
            FOCUSED_ID=$(niri msg -j focused-window | jq '.id' 2>/dev/null)

            IMV_WINDOW_ID=$(niri msg -j windows | jq ".[] | select(.pid == $IMV_PID) | .id" 2>/dev/null)

            if [ -n "$IMV_WINDOW_ID" ] && [ "$FOCUSED_ID" != "$IMV_WINDOW_ID" ]; then
                kill $IMV_PID
                break
            fi
            sleep 0.2
        done
    ) &

elif command -v xdg-open >/dev/null 2>&1; then
    xdg-open "$FILE" &
fi
```

![imv](https://github.com/user-attachments/assets/eb8ce006-7741-4844-9f5a-1ce4b07a4a95)

## ocr

Copy text from selected image file using `tesseract`.

**Requires `tesseract`, `tesseract-data-eng` (or other languages), and `wl-clipboard` to be installed.**

```bash
#!/usr/bin/env bash
FILE=$(echo -n "$NAUTILUS_SCRIPT_SELECTED_FILE_PATHS" | head -n 1)
[ -n "$FILE" ] || exit 0

tesseract "$FILE" stdout -l eng | wl-copy

MIME_TYPE=$(file --mime-type -b "$FILE")

if [[ "$MIME_TYPE" == image/* ]]; then
    notify-send "OCR Complete" "Text from $(basename "$FILE") copied to clipboard."
fi
```

![ocr](https://github.com/user-attachments/assets/95657430-1420-4169-9de6-e1f8b9e9585d)

# Setup Specialized Versions
## Niri
I am currently using a Niri setup, these are the scripts specialized for the specific setup. They should work on your Niri setup too out-of-the-box or with some small adjustments.
Refer to the: [niri](/niri/)
### Differences
- Their window automatically close with this `niri` config:
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

window-rule {
    match app-id="dev.ghostty.preview"
    open-floating true
    default-column-width { proportion 0.4; }
    default-window-height { proportion 0.6; }
}
```
