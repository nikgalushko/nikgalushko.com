+++
author = "Nikita Galushko"
title = "Разбираем go:embed в Go 1.16"
date = "2021-01-22"
description = "Встраиваем файлы в бинарник на go."
tags = [
    "go",
]
categories = [
    "go",
]
+++
Go 1.16 официально еще не вышел, но уже сейчас можно скачать бету с официального сайта и поиграться. Этим и займемся. В этой статье разберем работу нового пакет `embed`.
## #0 устанавливаем beta go 1.16
Если у вас установлен Go, то установка беты происходит максимально просто. В терминале выполняем последовательно следующие команды.
```sh
go get golang.org/dl/go1.16beta1
go1.16beta1 download
```
Вот и все, теперь бета доступна посредством вызова `go1.16beta1`.
## embed
Что если мы хотим встроить файл в наш бинарник на go, например, какие-то шаблоны, html файлы, если это веб сервер или даже README.md ? Нам приходилось либо саморучно затаскивать их в наш код, либо пользоваться сторонними библиотечками, такими как [go-bindata](https://github.com/go-bindata/go-bindata). Оба варианта отстойные. Первый способ не гибкий, в нем можно ошибиться, так как нужно все делать вручную. Второй способ получше, но это дополнительные зависимости, которых может не оказаться в вашей среде и главное дополнительные шаги при сборке приложения.

Go 1.16 решает нашу проблему директивой `//go:embed path_pattern`.

## Условия использования директивы
- директива должна предшествовать строке, содержащей объявление переменной, в которую будет помещен файл. Между директивой и объявлением переменной допускаются только пустые строки и комментарии

- паттерн пути к файлу или директории не должен начинаться с `/` и иметь в себе `.` или `..`

- паттерн должен соответствовать хотя бы одному файлу или не пустой директории. В противном случае сборка не состоится

- симлинки запрещено использовать в паттерне

- паттерн может принимать только файлы или директории внутри модуля, но не во вне

- чтобы получить все файлы в директории нужно использовать `*`

## Встраиваем файл
Директива `//go:embed` позволяет нам встроить файла как строку `string`, так и как слайс байт `[]byte`.
```go
package main

import (
	_ "embed"
	"log"
)

//go:embed README.md
var readme string

//go:embed bkg.png
var image []byte

func main() {
	log.Print(readme)
}
``` 

В данном примере файлы располагаются следующим образом:
```sh
.
├── README.md
├── bkg.jpeg
└── main.go
```

Теперь содержимое файла README.md лежит в перменное `readme`, а содержимое bkg.png в переменной `image`. При этом это обычные переменные, которые мы можем менять в ходе выполнения нашей программы. 

## Встраиваем несколько файлов aka embed.FS
Мы поняли как встраивать один файл, но что делать, если у нас директория с несколькими html файлами, а еще директория с изображениями. Как нам встроить это все в наш бинарник ?

На этот раз наш пакет будет выглядит так:
```sh
.
├── README.md
├── bkg.jpeg
├── main.go
└── www
    ├── html
    │   ├── about.html
    │   └── index.html
    └── images
        ├── forest.jpg
        └── snow_forest.jpg
```
А код, которые встраивает в себя всю директорию www следующий:
```go
package main

import (
	"embed"
	"log"
)

//go:embed README.md
var readme string

//go:embed www
var www embed.FS

func main() {
	log.Print(readme)

	entries, err := www.ReadDir("www/html")
	if err != nil {
		log.Fatal(err)
	}

	for _, entry := range entries {
		log.Println(entry.Name())
	}
}
```

При этом `embed.FS` реализует интерфейс [fs.FS](https://tip.golang.org/pkg/io/fs/#FS), что очень удобно для абстрагирования в коде откуда на самом деле он читает файлы.
Для `embed.FS` есть ряд ограничений:

- это строго read-only структура, так что можно свободно передавать ее в горутины

- паттерн заканчивающийся на `/*` встраивает все файлы даже те, которые начинаются на `.` и на `_`

## Пару слов напоследок
В конце хочу поделиться еще маленьким нюансом, который заметил. Встраивание двух единичных одинаковых файлов будет честным, то есть, если мы делаем что-то подобное:
```go
//go:embed bkg.jpeg
var image []byte

//go:embed bkg.jpeg
var image2 []byte
```
то размер файла увеличится на 2 размера файла bkg.jpeg. При этом для встраивания через `embed.FS` это не так.