---
title: "Fish Shell: Shell Interaktif yang User-Friendly"
date: "2026-04-01T10:30:00+07:00"
#menu: "main"
#
# description is optional
#
# description: "Pengenalan Fish Shell sebagai alternatif modern untuk Bash dan Zsh"
slug: "fish-shell"
tags: ["terminal","shell","linux","productivity"]
---

Jika selama ini kita terbiasa pakai Bash atau Zsh, kita mungkin belum mengenal Fish Shell, sebuah shell modern yang dirancang khusus untuk kemudahan penggunaan dan pengalaman interaktif yang lebih baik. Kalau kita jujur, Bash dan Zsh itu powerful tapi konfigurasinya bikin pusing, Fish Shell hadir dengan fitur-fitur canggih yang langsung bisa dipakai tanpa perlu konfigurasi rumit.

Fish adalah kependekan dari "Friendly Interactive SHell", dan nama tersebut benar-benar mencerminkan filosofinya. Shell ini dirancang dengan prioritas pada keramahan pengguna, dengan syntax yang lebih intuitif dan fitur-fitur yang berguna langsung ada dari dalam kotaknya.

> [Fish Shell - Official Website](https://fishshell.com/)

### Kelebihan Fish Shell

- **Syntax yang Intuitif** - Syntax Fish lebih mirip dengan bahasa pemrograman modern, bukan legacy shell scripting
- **Autocompletion yang Cerdas** - Suggestion muncul otomatis berdasarkan history dan deskripsi perintah
- **Man Pages Integration** - Autocompletion langsung dari man pages, jadi kita tau apa yang tiap parameter lakukan
- **Warna dan Formatting Otomatis** - Syntax highlighting bawaan untuk command dan error
- **History yang Better** - Cari history dengan prefix matching atau full-text search
- **Vi Keybindings Built-in** - Bisa pakai vi mode tanpa perlu konfigurasi ekstensif
- **Config yang Mudah** - Web-based configuration GUI atau simple config file

### Installasi

Tergantung distribusi Linux yang kita gunakan:

**Ubuntu/Debian:**
```bash
sudo apt install fish
```

**Fedora/RHEL:**
```bash
sudo dnf install fish
```

**macOS:**
```bash
brew install fish
```

Setelah install, set Fish sebagai default shell:
```bash
chsh -s /usr/bin/fish
```

### Pengalaman Pertama

Saat pertama kali membuka Fish, kita akan langsung merasakan perbedaannya. Ketik beberapa karakter perintah, dan Fish akan otomatis memberikan suggestion dengan warna abu-abu di belakang cursor. Tekan Tab atau Right Arrow untuk menerima suggestion tersebut.

```fish
> ls -
# Fish akan suggest: -A, -a, -B, -b, dll beserta penjelasan singkatnya
```

Coba juga search history dengan Ctrl+R atau Ctrl+S:

```fish
> <Ctrl+R>
# Akan membuka history search yang interactive
```

### Fitur Autocompletion

Fish ini hebat soal completion. Misalnya kita pakai Docker, saat kita ketik:

```fish
docker run -i
```

Fish akan otomatis suggest:
- `-i` (--interactive)
- `--image`
- `--ipc`

Beserta penjelasan singkat tiap option, langsung dari man pages.

### Konfigurasi Dasar

Config Fish disimpan di `~/.config/fish/config.fish`. Syntax-nya simple dan readable:

```fish
# Set environment variable
set -gx EDITOR vim

# Create alias
alias ll 'ls -lah'
alias gs 'git status'

# Create function
function greet
    echo "Halo, $argv!"
end
```

Kita juga bisa gunakan web-based configuration:

```bash
fish_config
```

Perintah ini akan buka browser dengan GUI untuk edit theme, prompt, keybindings, dll.

### Function vs Alias

Di Fish, function lebih powerful dari alias. Contoh membuat function yang bener:

```fish
function cd_and_list
    cd $argv[1]
    ls -lah
end
```

Function Fish mendukung arguments, loops, conditions, dan logic yang lebih complex dibanding alias.

### Prompt Customization

Prompt Fish bisa di-customize dengan simple function:

```fish
function fish_prompt
    set_color brgreen
    echo -n (whoami)
    set_color normal
    echo -n '@'
    set_color blue
    echo -n (hostname)
    set_color normal
    echo -n ':'
    set_color yellow
    echo -n (pwd)
    set_color normal
    echo -n ' > '
end
```

### Private Mode

Fish punya fitur private mode yang mencegah command tersimpan di history:

```bash
fish --private
```

Atau gunakan Ctrl+X di dalam Fish untuk toggle private mode.

### Integration dengan Tools

Fish mudah diintegrasikan dengan tool modern seperti fzf, direnv, atau environment manager lainnya. Syntax-nya lebih clean dan readable dibanding bash:

```fish
# Activate fzf keybindings
fzf_key_bindings

# Use with direnv
eval (direnv hook fish)
```

### Saat Kita Perlu Script

Satu hal yang perlu diingat, Fish bukan POSIX-compliant shell. Jika kita perlu menulis production script yang harus portable, tetap gunakan Bash atau Sh. Tapi untuk interactive use dan scripting lokal, Fish sangat cocok.

### Kesimpulan

Fish Shell adalah pilihan yang bagus untuk siapa saja yang ingin terminal experience yang lebih interactive dan user-friendly. Tidak perlu konfigurasi rumit, fitur-fitur canggih langsung tersedia, dan syntax-nya jauh lebih intuitif. Kalau kita sering main di terminal dan ingin produktivitas lebih baik, cobalah Fish Shell. Paling tidak untuk beberapa hari dan rasakan perbedaannya sendiri.

Pengalaman saya pribadi, setelah switch ke Fish, saya tidak pernah balik ke Bash untuk interactive shell. Productivity meningkat karena completion yang smart dan feature-rich, dan honestly, terminal jadi lebih menyenangkan untuk digunakan.
