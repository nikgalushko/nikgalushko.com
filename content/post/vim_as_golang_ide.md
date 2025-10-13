+++
author = "Nikita Galushko"
title = "Делаем из Vim IDE для Golang"
date = "2021-02-02"
description = "В этой статье я пошагово покажу как из neovim сделать отличную IDE под Golang. Пошагово пройдем по установке и настройке всего необходимого."
tags = [
    "go",
    "vim"
]
categories = [
    "go",
]
+++
## Предисловие
Уверен, у кого-то возникнет вопрос: «Зачем в 2021-ом пользоваться -vim- neovim, когда есть замечательный [Goland](https://www.jetbrains.com/go/)?». Я до какого-то времени сам пользовался только продуктами от JetBrains, но вот уже более 1.5 лет пишу исключительно в vim. 

Основной причиной для моего перехода с Goland стала его прожорливость. Отдавать по 3-4 Гб программе, которая просто подсказывает мне названия методов, запускает тесты и реализует «Go to definition», очень расточительно. Вкупе с непонятной политикой индексации проекта, когда у тебя в момент взывают куллеры, а батарейка на глазах тает, привело меня к vim.

## #1
Наша сборка будет зиждиться на двух китах: [vim-go](https://github.com/fatih/vim-go) и [coc.nvim](https://github.com/neoclide/coc.nvim), как LSP клиент к [gopls](https://github.com/golang/tools/tree/master/gopls). Vim-go будет отвечать за сборку нашего проект, запуск тестов, форматирование кода, отладку, Go Docs. Coc.nvim + gopls будут отвечать за реализацию «Go to definition», автокомплит, переименование, короче за все то, что из простого редактора делает удобный в использовании инструмент. 

## Установка необходимых зависимостей
##### #0 Neovim
Чтобы настроить neovim его необходимо сначала поставить 😄. Для macOS, а именно на этой системе я работаю, установка происходит в одну команду:
```sh
brew install neovim
``` 

После установки стоит создать файл `~/.config/nvim/init.vim`. Это аналог `.vimrc`, в нем происходит вся настройка neovim.

`brew` это пакетный менеджер. Более подробно о том, как его установить можно прочить на [официальном сайте](https://brew.sh/index_ru).

##### #1 vim-plug
[vim-plug](https://github.com/junegunn/vim-plug) — менеджер плагинов для vim/neovim. Установка проста:
```sh
curl -fLo ~/.local/share/nvim/site/autoload/plug.vim --create-dirs \
    https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
```

##### #2 yarn
[yarn](https://yarnpkg.com/) необходим coc.nvim для автокомплита. Установка, как и в случае с vim-plug в одну команду:
```sh
npm install --global yarn
```

## Установка vim-go
И так, мы установили все необходимы зависимости, а следовательно, готовы перейти к главной части. Сначала установим vim-go.

Все очень просто. Открываем созданный нами ранее файл `~/.config/nvim/init.vim` и вписываем в него следующий текст:
```
call plug#begin('~/.vim/plugged')
Plug 'fatih/vim-go'
call plug#end()
```
В этом блоке мы описываем те плагины, которые хотим установить в neovim, мы еще не раз к нему вернемся.
После изменения `~/.config/nvim/init.vim` в консоле выполняем последовательно две команды:
```
nvim +PlugInstall
nvim +GoInstallBinaries
```

## Установка coc.nvim
Заходим в `~/.config/nvim/init.vim` и добавляем `Plug 'neoclide/coc.nvim', {'do': 'yarn install --frozen-lockfile'}` после строки `Plug 'fatih/vim-go'`. На данный момент ваш init.vim файл должен выглядит так:
```
call plug#begin('~/.vim/plugged')
Plug 'fatih/vim-go'
Plug 'neoclide/coc.nvim', {'do': 'yarn install --frozen-lockfile'}
call plug#end()
```
Устанавливаем плагин командой `nvim +PlugInstall`.

После установки рекомендую дополнить свой init.vim файл для начала [дефолтной конфигураций](https://github.com/neoclide/coc.nvim/tree/master#example-vim-configuration) coc.nvim. Также стоит добавить в init.vim следующую строчку, которая отключает «Go to definition» через сочетание `gd` в vim-go, мы же хотим, чтобы этим занимался gopls:
```
let g:go_def_mapping_enabled = 0
```

Чтобы проверить, что все установилось правильно зайдите в neovim и выполните `:CocInfo`. Команда должна вывести что-то похожее на:
```sh
## versions

vim version: NVIM v0.4.4
node version: v14.14.0
coc.nvim version: 0.0.79
coc.nvim directory: /Users/nikita.galushko/.vim/plugged/coc.nvim
term: xterm-kitty
platform: darwin
```

##### Настройка coc.nvim
Для настройки coc.nvim создайте файл `~/.config/nvim/coc-settings.json` со следующим содержимым:
```json
{
  "languageserver": {
    "golang": {
      "command": "gopls",
      "rootPatterns": ["go.mod", ".vim/", ".git/"],
      "filetypes": ["go"],
      "initializationOptions": {
        "usePlaceholders": true
      },
      "gopls": {
        "semanticTokens": true
      }
    }
  }
}
```

## Последние штрихи
Сейчас у нас готовый плацдарм для работы с проектом на Golang, есть автокомплит, запуск тестов. Для полноты не хватаете всего пару вещей.
##### Устанавливаем nerdtree
Чтобы удобно работать необходимо иметь возможность видеть дерево проекта. В этом нам поможет [nerdtree](https://github.com/preservim/nerdtree). Добавляем плагин в init.vim `Plug 'preservim/nerdtree’` и устанавливаем `nvim +PlugInstall`. Приправим nerdtree небольшой настройкой в init.vim:
```
" запуск nerdtree автоматически
autocmd StdinReadPre * let s:std_in=1
autocmd VimEnter * if argc() == 0 && !exists("s:std_in") | NERDTree | endif
" закрываем neovim если открыт только nerdtree
autocmd bufenter * if (winnr("$") == 1 && exists("b:NERDTree") && b:NERDTree.isTabTree()) | q | endif
```
##### А как же глобальный поиск?
Иногда «Go to definition» недостаточно и нужен поиск по всему проекту. В этом нам поможет [fzf](https://github.com/junegunn/fzf.vim). В init.vim добавляем плагин:
```
Plug 'junegunn/fzf', { 'do': { -> fzf#install() } }
Plug 'junegunn/fzf.vim'
```
Устанавливаем уже знакомой командой `nvim +PlugInstall`. Остался последний шаг установить [the_silver_searcher](https://github.com/ggreer/the_silver_searcher). Тут тоже все просто: `brew install the_silver_searcher`. Теперь мы можем искать по всему проекту простой командой `:Ag <то что хотим найти>`.

-----

**Автор**: Никита Галушко - Golang инженер, специализирующийся на производительности и распределённых системах.
Больше технических разборов в Telegram: https://t.me/b1tw1se