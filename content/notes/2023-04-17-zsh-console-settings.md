---
title: "Конфигурирование ZSH под iTerm2"
slug: "zsh-console-settings"
author: ""
type: ""
date: 2023-04-17T12:00:00+03:00
subtitle: ""
image: ""
tags: []
private: false
---
В заметке собраны мои основные настройки для zsh и iTerm: темы, утилиты, powerlevel10k, .zshrc.

<!--more-->

# iTerm2
* [iTerm2](https://iterm2.com) - приложение для MacOS
* [cobalt2](https://iterm2colorschemes.com) - тема для iTerm
* [nerdfonts](https://www.nerdfonts.com/font-downloads) - пропатченные шрифты с символами (для powerlevel10k)

# CLI утилиты
* [brew](https://brew.sh) - менеджер пакетов для MacOS
* [sdkman](https://sdkman.io) - управление java sdk и утилитами
* [lsd](https://github.com/lsd-rs/lsd) - красивая версия ls
* [ripgrep (rg)](https://github.com/BurntSushi/ripgrep) - красивая версия grep

# ZSH
* [oh-my-zsh](https://ohmyz.sh) - фреймворк для конфигурации ZSH
* [zap](https://github.com/zap-zsh/zap) - менеджер плагинов для zsh (еще не пробовал, но выглядит удобно)
* [powerlevel10k](https://github.com/romkatv/powerlevel10k) - модификация строки ввода

## .zshrc
Базовые настройки из файла `.zshrc`:
```bash
### ZSH
# Тема
ZSH_THEME="powerlevel10k/powerlevel10k"

# Плагины
plugins=(git sdk zsh-autosuggestions zsh-syntax-highlighting zsh-history-substring-search)

# Отключает все элементы справа в 
# Вставлять после source ~/.p10k.zsh
POWERLEVEL9K_RIGHT_PROMPT_ELEMENTS=()

# Настройка внешней клавиатуры для плагина zsh-history-substring-search
bindkey "$terminfo[kcuu1]" history-substring-search-up
bindkey "$terminfo[kcud1]" history-substring-search-down

### Алиасы
# Быстрый переход в dev папку
alias dev="cd ~/dev"

# Использованть lsd вместо ls
alias ls="lsd"
alias lst='ls --tree'
alias lsa='ls -la'

### Git
# Переход на ветку main с checkout
alias main='git checkout main && git pull --rebase origin main'
```
