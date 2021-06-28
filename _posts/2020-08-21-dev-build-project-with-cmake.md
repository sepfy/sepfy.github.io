---
layout: post
title: 程式開發 - 使用CMake建置專案
date: 2020-08-21
categories: Programming
---

在做Linux程式開發時, 常常使用Makefile來幫助我們建置專案, 而CMake是一個幫助我們建立跨平台專案的工具, 可以想像成用來幫我們在Linux產生Makefile, 在Windows產生Visual Studio Solution的工具. 最近單純拿來當作Makefile的替代品覺得也不錯, 所以記錄了一些實用的功能
# 第一支執行檔
假設我們開發了一個名叫sample的專案, 目錄的結構大致如下:
```
- sample/
    - CMakeLists.txt 
    - src/ 
        - CMakeLists.txt
        - main.c
```
如果要利用CMake幫助我們建構專案, 我們需要在目錄下新增CMakeLists.txt, 並在檔案中加入相關的建置訊息, 以下是一個基本的範例, 放置在專案的根目錄, cmake_minimum_required代表使用的CMake版本最低需求以及加入src這個路徑.
```cmake
cmake_minimum_required(VERSION 3.1)
project(samples)

add_executable(main main.c)
```

當CMake在建置專案的時候, 就會去尋找src目錄下的CMakeLists.txt檔案, 這裡我們加入一個最簡單的小程式main.c並把它編譯成執行檔. project用來設定當前子專案的名稱, 因為一個專案可能由多個子專案建置而成, 專案名稱可以幫助我們連結到正確的目錄. add_executable則是將相關的程式碼編譯成執行檔, 如果有多個檔案可以繼續接在main.c之後並用空白隔開.
cmake_minimum_required(VERSION 3.1)
add_subdirectory(src)

接著只要使用大家熟悉的CMake指令就可以將main程式編譯出來.
```bash
$ cd sample
$ mkdir cmake
$ cmake ..
$ make
```
# 編譯多個檔案
如果你有很多檔案都需要各自編譯成執行檔, 以下範例可以將該目錄底下的所有.c檔各自編譯成執行檔, 如果是要合併成一支執行檔, 可以直接使用${APP_SOURCES}變數.
```cmake
cmake_minimum_required(VERSION 3.1)
project(app)

file(GLOB APP_SOURCES "*.c")

foreach(sourcefile ${APP_SOURCES})
    string(REPLACE ".c" "" appname ${sourcefile})
    string(REPLACE "${PROJECT_SOURCE_DIR}/" "" appname ${appname})
    add_executable(${appname} ${sourcefile})
endforeach(sourcefile ${APP_SOURCES})
```

# 安裝路徑
一般編譯出來的檔案都會放在cmake的建置路徑下, 如果想要將程式安裝到指定的目錄, 可以加入install的設定.
```cmake
cmake_minimum_required(VERSION 3.1)
project(samples)

add_executable(main main.c)
install(TARGETS main
    DESTINATION ${CMAKE_INSTALL_PREFIX}/bin/
)
```
CMAKE_INSTALL_PREFIX是一個預設的變數, 如果沒有指定, 預設為系統根目錄, 上述範例就會安裝到/bin/底下, 也可以在建置時指定路徑
```bash
$ cmake -DCMAKE_INSTALL_PREFIX=<your path> ..
```

# 連結函式庫
一個專案除了執行檔以外, 我們也可能將部份程式碼編譯成函式庫的方式供不同的程式使用, 假設我們的專案目錄增加了一個名為func的子專案, 而func.c提供了一些可用的函式.
```
- sample/
    - CMakeLists.txt 
    - src/ 
        - CMakeLists.txt
        - main.c
        - func/
            - CMakeLists.txt
            - func.h
            - func.c
```
我們先來看func下的CMakeLists.txt, 類似前面的主程式, 只是把add_executable改為add_library而已, 這樣將會產生一個名為libfunc.a的靜態函式庫
```cmake
cmake_minimum_required(VERSION 3.1)
project(func)

add_library(func func.c)
```

如果要在main.c中引用它, 可以參造以下方法, include_directories代表指定header檔的位置, 在func下的CMakeLists.txt中我們設定了project名稱為func, 這樣我們就可以利用${func_SOURCE_DIR}來獲取func專案的位置. target_link_libraries則是加入我們要連結的函式庫名稱, 例如我們自己寫的libfunc, 或者使用系統的函式庫, 例如pthread
```cmake
cmake_minimum_required(VERSION 3.1)
project(samples)
add_subdirectory(func)


add_executable(main main.c)
# Install target
install(TARGETS main
    DESTINATION ${CMAKE_INSTALL_PREFIX}/bin/
)


# Link libraries
include_directories(${func_SOURCE_DIR}/)
add_executable(main_func main_func.c)
target_link_libraries(main_func func pthread)
```

# 加入參數
假設我們想加入一個unittest程式, 但我們只有在開發時需要將它編譯出來, 可以增加一個選項來設定它
```
- sample/
    - CMakeLists.txt 
    - src/ 
        - CMakeLists.txt
        - main.c
        - func/
            - CMakeLists.txt
            - func.h
            - func.c
            - func_unittest.c
```
以下範例我們用option來設定一個UNITTEST_ENABLE的變數, 並透過if條件來判斷是否要編譯unittest程式, 中間我們加入了一個過濾器, 將有unittest命名的檔案獨立出來.
```cmake
cmake_minimum_required(VERSION 3.1)
project(func)

add_library(func func.c)


option(UNITTEST_ENABLE "Build the unittest code" OFF)

file(GLOB func_SRC "*.c")
file(GLOB unittest_SRC "*unittest*")

list(REMOVE_ITEM func_SRC ${unittest_SRC})

if(UNITTEST_ENABLE)
  add_executable(unittest ${unittest_SRC})
endif(UNITTEST_ENABLE)
```

使用方式如下
```sh
$ cmake -DUNITTEST_ENABLE=true ..
```

# 呼叫指令
某些時候我們可能需要呼叫指令幫我們產生一些設定檔或程式碼, 可以使用execute_process來執行相關指令, 以下範例會輸出”Hello command”的訊息.
```cmake
# Execute command
execute_process(COMMAND echo -e "\\033[0;33mHello command\\033[0m")
```
以上是自己在開發上使用到的一些基本功能, 將來有其他實用的功能再分享出來
