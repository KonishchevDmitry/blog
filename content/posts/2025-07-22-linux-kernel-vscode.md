---
title: Настраиваем VS Code для работы с кодом ядра Linux
slug: linux-kernel-vscode
date: 2025-07-22T09:30:09+03:00
---

Мне никогда не приходилось заниматься разработкой ядра, но вот в последнее время всё чаще возникает необходимость заглянуть в его исходники, чтобы уточнить для себя, как именно работает тот или иной системный вызов или файл в sysfs/proc. И каждый раз это было жутко неудобно, т. к. IDE сходу не могло адекватно проиндексировать код ядра, чтобы можно было более или менее сносно прыгать по функциям. Поэтому решил потратить какое-то время и разобраться, как можно улучшить эту ситуацию.

## C/C++ extension

Если погуглить, то наиболее частая рекомендация сводится к использованию VS Code с расширениями [C/C++](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools) и [Makefile Tools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.makefile-tools) с примерно следующим `.vscode/c_cpp_properties.json`:

```json
{
    "configurations": [
        {
            "name": "Linux",

            "includePath": [
                "${workspaceFolder}/arch/x86/include/generated",
                "${workspaceFolder}/arch/x86/include",
                "${workspaceFolder}/include",
                "${workspaceFolder}/**"
            ],
            "forcedInclude": [
                "${workspaceFolder}/include/generated/autoconf.h"
            ],

            "dotConfig": "${workspaceFolder}/.config",
            "configurationProvider": "ms-vscode.makefile-tools",
            "compileCommands": "${workspaceFolder}/compile_commands.json",

            "compilerPath": "/usr/bin/gcc",
            "intelliSenseMode": "linux-gcc-x64",

            "cStandard": "gnu11",
            "cppStandard": "gnu++11"
        }
    ],
    "version": 4
}
```

В итоге оно работает, но очень ограниченно: навигация по коду постоянно тормозит и либо вовсе не находит часть символов, либо для части функций находит только их объявление в заголовочном файле, но не реализацию – и нормально работать в таком режиме просто невозможно, т. к. постоянно приходится переключаться на обычный поиск по содержимому файлов.

## clangd

Но, как оказалось, у вышеупомянутых расширений от Microsoft есть очень достойная альтернатива в виде расширения [clangd](https://marketplace.visualstudio.com/items?itemName=llvm-vs-code-extensions.vscode-clangd), которое работает поверх полноценного language server'а [clangd](https://clangd.llvm.org/). И это очень хорошая альтернатива, особенно учитывая то, что с относительно недавних пор ядро Linux [поддерживает](https://docs.kernel.org/kbuild/llvm.html) сборку clang'ом.

И оно действительно **работает**. Да – первоначальное построение индекса занимает какое-то время, но вот зато потом IDE видит все символы и, что не менее важно, осуществляет очень быструю навигацию по ним. Проблемы возникают разве что с заголовочными файлами, для которых в силу понятных причин не может быть однозначной информации, с какими флагами компиляции они будут использоваться.

## Приступаем к работе

Клонируем репозиторий:

```bash
git clone git@github.com:torvalds/linux.git
```

Чтобы самому не возиться с конфигурацией ядра, а также работать именно с той, которая будет использоваться в реальной жизни, берём конфигурацию из текущей системы:

```bash
cp "/boot/config-$(uname -r)" .config
```

Устанавливаем пакеты, которые нам понадобятся для сборки:
* Fedora: `sudo dnf install bc bison clang clangd elfutils-libelf-devel flex lld llvm make ncurses-devel openssl-devel xz zstd`
* Ubuntu: `sudo apt install bc bison clang clangd flex libelf-dev libncurses-dev libssl-dev lld llvm make xz-utils zstd`

В `.vscode/settings.json` задаём общие настройки проекта:

```json
{
    "[c]": {
        "editor.tabSize": 8,
        "editor.insertSpaces": false,
        "editor.detectIndentation": false,
        "editor.rulers": [80, 100]
    },
    "files.associations": {
        "*.h": "c"
    },

    "files.exclude": {
        "**/modules.order": true,
        "**/.*.*.cmd": true,
        "**/*.a": true,
        "**/*.o": true,
        "**/*.ko": true,
        "**/*.mod": true,
        "**/*.mod.c": true,
        "**/*.symvers": true
    }
}
```

Далее устанавливаем расширение [clangd](https://marketplace.visualstudio.com/items?itemName=llvm-vs-code-extensions.vscode-clangd) – либо через интерфейс VS Code, либо с помощью команды `code --install-extension llvm-vs-code-extensions.vscode-clangd`.

И я очень рекомендую добавить следующий параметр в настройки VS Code, в котором после `-j` указать количество ядер в вашем процессоре, т. к. по какой-то причине clangd по умолчанию при индексации использует только часть из них, что на таком большом проекте, как ядро Linux, приводит к невероятно долгой первичной индексации:

```json
{
    "clangd.arguments": ["-j", "10"]
}
```

clangd ожидает, что в корне проекта у нас будет файл [compile_commands.json](https://clang.llvm.org/docs/JSONCompilationDatabase.html), в котором будут указаны опции компиляции каждого `*.c`-файла проекта. Именно благодаря этому файлу он может правильно проиндексировать код. Поэтому запускаем следующую команду:

```bash
make -j "$(nproc)" LLVM=1 compile_commands.json
```

... и идём заниматься своими делами – работать оно будет довольно долго.

Как только `make` отработает, и у нас появится `compile_commands.json`, открываем в VS Code любой `*.c`-файл, чтобы стриггерить clangd – и он тут же начнёт индексировать наш проект (в status bar будет соответствующая информация о прогрессе индексации). Тут тоже придётся подождать какое-то время, но только в первый раз – при последующем переоткрытии проекта оно будет подхватываться гораздо быстрее.

Как только индексация закончится, должны заработать все стандартные средства навигации по коду.

## Работа с кодом ядра из MacOS

В качестве рабочего инструмента у меня MacBook Pro M1 и, казалось бы, это не самая удобная конфигурация для того, чтобы копаться в исходниках ядра, но, к счастью, это не так.

Устанавливаем [OrbStack](https://orbstack.dev/) и создаём себе виртуальную машину:

```bash
orbctl create ubuntu kernel
```

При этом, несмотря на то, что OrbStack поддерживает эмуляцию x86 через [Rosetta](https://developer.apple.com/documentation/apple-silicon/about-the-rosetta-translation-environment), в ней нет необходимости, т. к. при сборке ядра мы можем использовать кросс-компиляцию.

Заходим на нашу виртуальную машину:

```bash
orb
```

и клонируем наш репозиторий.

Добавляем в самое начало `~/.vscode/ssh/config` следующую строку:
```
Include ~/.orbstack/ssh/config
```

А затем с помощью расширения [Remote - SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh) подключаемся VS Code'ом к нашей виртуальной машине и выполняем все действия, которые были перечислены в предыдущем разделе, но за одним исключением – при вызове `make` необходимо указать `ARCH=x86`, чтобы наша ARM'овая виртуалка собрала ядро под целевую архитектуру (x86-64):

```bash
make -j "$(nproc)" LLVM=1 ARCH=x86 compile_commands.json
```
