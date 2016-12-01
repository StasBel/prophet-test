# Prophet test

- [TODO](#todo)
- [Prereq](#prereq)
- [Files](#files)
- [Commands](#commands)
- [Flags](#flags)
- [Examples](#examples)
- [Running](#running)
- [Docker](#docker)
- [Benchmarks](#benchmarks)

## TODO

- [x] Разобраться с ошибкой запуска clang'ом c-шного кода.
Вероятней всего, ошибка связана с трюком с symlink'ами:

    ```
    In file included from square-bug.c:2:
    /usr/include/stdio.h:33:11: fatal error: 'stddef.h' file not found
    # include <stddef.h>
    ``` 

- [ ] Добавить autocomplete для *build* скрипта в docker-образе.

## Prereq

1. Убеждаемся, что Prophet собирается и запускается по инструкции Тимофея c почты.
2. Нам потребуется *clang-3.6*. Убеждаемся, что он есть и работает, а если нет:

    ```
    sudo apt-get install clang-3.6
    ```

3. Нам также потребуется *gcc*:

    ```
    sudo apt-get install gcc
    ```

4. Идем в */prophet-gpl/wrap* и собираем с помощью *make*.
Убеждаемся, что в папке нет файлов *gcc*, *cc* или *g++*.
Тут же в файле *pclang.py* примерно на 152 строчке (после того, как мы пытаемся получить значение из env) вставляем hardcode:

    ```python
    clang_cmd = "/usr/bin/clang-3.6"
    ```

5. Также перемещаем shebang на первую строчку во всех .py файлах:

    ```python
    #!/usr/bin/env python
    ```

6. Идем в */prophet-gpl/include* и собираем c помощью *make*.
Результатом должен быть файл *_prophet_profile.h*.
7. Проверьте и исправьте, если нужно строчки с 4 по 15 в *config.h* (префикс wrap_path будет свой):

    ```c
    /* the clang cmd full path */
    #define CLANG_CMD "/usr/bin/clang"
    
    /* the location of the wrapper for instrument the file */
    #define CLANG_WRAP_PATH "/home/ubuntu/prophet-gpl/wrap"
    
    /* the extra include path arguments that need to pass to clang when build AST
       tree */
    #define EXTRA_CLANG_INCLUDE_PATH ""
    
    /* the gcc cmd full path */
    #define GCC_CMD "/usr/bin/gcc"
    ```

8. Пересоберите Prophet с помощью *make*.
9. Осталось только написать пару скриптов и все правильно запустить.

## Files

Какие файлы нам понадобятся для запуска:

1. Исходники на C. Хотя бы один файл с ошибкой.
2. Тесты. Хотя бы на одном программа выдает правильный результат и хотя бы на одном неправильный.
3. \*.revlog файл, где перечислены номера правильных и неправильных тестов (пример в *mytest/square.revlog*).
4. *tester_common.py* файл, просто скопируйте его из *mytest*, там нам понадобится одна функция.
5. У вас наверняка будет *Makefile*.
!!!ВАЖНО: компиляция и линковка должны быть обязательно разделены. И лучше чтобы все это делалось gcc:

    ```makefile
    gcc -c file.c -o file.o
    gcc file.o -o file
    ```

6. \*-build.py скрипт, собирающий проект.
На выхода - результат запуска *extract_arguments* из *tester_common.py*.
Важно чтобы поддерживались следующие флаги и аргументы:
    
    ```
    *-build.py [-c] [-d file] src [out] : src, args
    ```
    flag/arg | def
    ------------ | -------------
    -c | только скомпилировать
    -d | dryrun_src, то, что нужно исключить из рассмотрения при выборе аргументов из лога компиляции
    src | собственно папка проекта, abspath
    out | скрипт парсит аргументы из лога компиляции и записывает в файл, если такой есть

7. \*-test.py скрипт, собирающий проект. На выходе - номера положительных тестов
Важно чтобы поддерживались следующие флаги и аргументы:
    
    ```
    *-test.py [-p src] src test work id+ : id*
    ```
    flag/arg | def
    ------------ | -------------
    -p | этот флаг передается когда тестируем с профайлером, фактически заменяет src
    src | src папка проекта, abspath
    test | папка с тестами, abspath
    work | рабочая папка, мною не использовалась
    id+ | список номеров тестов
    
8. \*.conf - конфигурационный файл, где указаны все пути ко всем нужным файлам,
а также название файла с ошибкой (не обязательно)

## Commands

Какие команды запускают вычисления:
    
    prophet -feature-para feature conf
    
arg | def
------------ | -------------
prophet | исполняемый файл prophet
feature | файл с параментами из обучения (я использовал *crawler/para-all.out*)
conf | конфигурационный файл

Еще можно приписать множество всевозможных флагов, а также разделить этапы локализации ошибки,
генерации патчей, их ранжирования или вообще использовать другой алгоритм.

## Flags

все: *prophet --help*

важные:

flag | def
------------ | -------------
-consider-all | рассматриваем все файлы, а не только bugged_file из *.conf файла
-full-explore | заканчиваем вычисления только когда испробовали все в search space
-full-synthesis | перебираем все условия вместо выбора первого
-naive | запускаем наивное исправление кода, только удаляем и вставляем выражения
-random | рандомный выбор кандидатов в search space
-stats | статистика

## Examples

###1. Вычисление квадрата числа с ошибкой:
Исходник:
```c
int calc(int arg) {
    if (arg != 487) {
        return arg * arg;
    } else {
        return arg + 1;
    }
}
```
Лучший патч (из 5):
```c
int calc(int arg) {
    //prophet generated patch
    if ((arg != 487) || (1)) {
        return arg * arg;
    } else {
        return arg + 1;
    }
}
```
И статистика:
```
Total 67 different repair schemas!!!!
Total 138 different repair candidate templates for scoring!!!
Total 102 different partial repair candidate templates!!
```
[Остальные патчи и прочая инфа.](https://github.com/StasBel/prophet-test/tree/master/results/square/)

## Running

Есть скрипт, чтобы быстро запустить вычисления на вашем проекте, не копаясь в *.conf* и *.revlog* файлах.
Пусть у вас есть папка с проектом *src_dir*, нужно соблюсти следующие условия:

1. Есть *Makefile* с раздельной компиляцией и линковкой.
2. Тесты лежат в *src_dir/tests*, если требуется их собрать (например, сгенерировать), то можно поставить там *Makefile*. Тесты должны быть вида *testчисло*.
3. В корневой папке *src_dir* должен лежать скрипт *run_test*, который по номеру теста выдает либо *positive* либо *negative*, в зависимости от того, правильный это тест или нет.

Тогда можно просто клонировать этот репозиторий и запустить *prophet* так:

```makefile
src/run src_dir
```

Также можно изменить пути к *prophet'у* и параметрам обучения в *src/Makefile*. Флаги *src/run*:

dep | def
------------ | -------------
-h, --help | print usage
-b, --bugged-file | если вы знаете, в каком файле ошибка, то его можно указать prophet'у (относительно *src_dir*)
-f, --full-serach | запускаем prophet с флагами -full-explore -full-synthesis -cond-ext
other | все остальные флаги передадутся prophet'у при запуске

Все результаты работы будут помещены в папку *workdir/* из папки, которой вы запускали *src/run*.

## Docker

Есть готовый docker-образ по имени *stasbel/prophet*, ~3.4G. Внутри - необходимое окружение для корректной сборки и запуска *prophet'a*. Собственно *prophet* уже собран и готов к запуску. Все *deps'ы* и файлы, необходимые для бенчмарков, удалены, чтобы образ весил меньше. Окружение, а также структура папок и файлов специально имитируют образ на *AWS*, для того чтобы все вычисления можно было проделать по инструкции с сайта *prophet*  и ничего при этом не изменяя. Ошибки со сборкой benchmark'ов пофикшены, правда проект *gmp* так и не удалось собрать, вылезает какая-то редкая ошибка c кодом на *c++* (что странно), поэтому *gmp* просто закомментирова в *Makefile'e* и вообще не собирается, все равно его вклад в результат не велик.

Быстрый запуск контейнера:

```
docker pull stasbel/prophet
docker run -it -u="ubuntu" -w="/home/ubuntu/" stasbel/prophet /bin/bash # user=ubuntu, pass=123
```

Описание важных файлов и папок:

file | def
------------ | -------------
benchmarks | файлы для benchmark'ов (deps'ы для сборки и тестов)
crawler | файлы для вытаскивания параметров обучения, поиска коммитов по гитхабу
tests/tools | скрипты для benchmark'ов
wrap | обертки проффилировщика для запуска компиляции и линковки на c, c++
src | исходники на c и исполняемые файлы
build | скрипт в помощь

Написал небольшой скрипт на питоне чтобы было удобней собирать и воспроизвести окружение для запуска benchmark'ов. Запуск: *./build [dep]*:

flg/arg | def
------------ | -------------
-h, --help | print usage
-b, --with-bench | флаг для install, сообщает, что зависимости в *benchmark/* также нужно собирать (по умолчанию - нет)
install | сборка prophet'a
bench-env | сборка окружения для запуска тестов из бенчмарков, размер контейнера после запуска увеличится до ~11G
clean | удаляем окружение и зависимости для бенчмарков
emacs-clean | удалить backup emacs'a
size | узнать текущий размер контейнера
git-test-repo/git-clean | скачать/удалить этот репозиторий

## Benchmarks

Успешно запускается на benchmark'ах, локализует ошибки с профайлером, генерирует патчи, но корректных почему-то нету. Запускал для *php-309516-309535*, *php-309579-309580*. Работает по 12 часов, выдает что-то типа:

```
The total number of synthesis runs: 0
The total number of concrete conds: 0
The total number of explored concrete patches: 32089
Repair process ends without working fix!!
Total 21016 different repair schemas!!!!
Total 27098 different repair candidate templates for scoring!!!
Total number of compiles: 10579
Total number of test eval: 28257
```

Цифры похожи по количеству патчей на результаты из статьи, но не выдающие никаких корректных патчей. Запустить на *AWS* теже вычисления не удалось, на бесплатной версии не хватает памяти (все крашится с ошибкой о *kernel memory*). Хотя можно уже отметить, что вместо заявленных *70 мин* это работало как минимум *8 часов* до падения (хотя и мощности меньше, но не во столько же раз).

Написал письмо разрабам. Ответили. Веду переписку.
