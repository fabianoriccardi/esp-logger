# ESP Logger for Arduino IDE

[![arduino-library-badge](https://www.ardu-badge.com/badge/ESP%20Logger.svg)](https://www.ardu-badge.com/badge/ESP%20Logger.svg) ![Compile Library Examples](https://github.com/fabiuz7/esp-logger-lib/actions/workflows/LibraryBuild.yml/badge.svg)

ESP Logger is an Arduino library offering a simple but complete interface to log events on ESP32 and ESP8266. Given the connection-oriented applications using these MCUs, ESP Logger provides a simple way to buffer data and efficiently send them through the most appropriate communication protocol.

## Motivations

Nowadays, tons of projects are related to the IoT world: this implies network connection and data collection. ESP8266 and ESP32 were born with native Wi-Fi paired with effective libraries to communicate. These libraries are very mature both in NONOS framework and in Arduino environment. Unfortunately, I noticed that often there is the need to rewrite everything about the data collection. For example, the existing libraries support perfectly raw files, but for every project, you should implement every basic control such as check if the file is open, decide the open mode, check lines terminator, check if the file is ended, check if there is enough space and so on. In my personal opinion, managing all these details every time is tedious and sloppy: avoiding them would be a plus feature in many projects. Raising the abstraction level is an unquestionable benefit for Makers interested in prototyping advanced connected objects. Moreover, abstractions always increase code sharing and reuse, without forgetting that it is a key point to promote communication among people, fundamental in Arduino-like communities. Before ending up with my library, I searched on the Internet to discover a library answering the requirements above. I didn't find anything matching all of them because, usually, data collection projects implement their custom solution, and often, searching for "log" term, you will find a lot of results regarding software development and debugging, not concrete data collection.

For all these reasons, I have developed ESP Logger library, which is built on top of flash memory and files, leaving you the decision on when logging, when flushing data, and how to flush data.

## Features

1. Log on internal flash (using LittleFS or SPIFFS) and or on SD card
2. Methods to monitor and limit log size
3. Support for multiple log files to store different information
4. Callback to flush your data over the network
5. Full control on when and how to flush your data
6. Agnostic to the data format, the atomic measurement unit is *Record*

## Installation

The latest version of ESP Logger is available on Arduino Library Manager and on [PlatformIO registry](https://platformio.org/lib/show/5879/ESP%20Logger).

## Usage

There are 2 main functions to be aware of: `append` and `flush`.

### Append

    bool append(const char* record)

it creates and stores a *Record*. You cannot log data containing non-printable characters, nor new line or carriage return. Return true if the data is saved, false otherwise.

### Flush

    bool flush()

it calls your callback function which has the following prototype:

    bool flusher(char* chunk, int n);

where *chunk* is a buffer that contains one or more records separated by '\0' char; *n* tells how many bytes is long the chunk, including '\0'. This function must return true if the flush process has succeeded, false otherwise (e.g. the server wasn't reachable). If true is returned, the library deletes the flushed data and, if other data are available, it calls `flusher()` again, otherwise it stops. If false, the library stops the flushing process and preserves the unflushed data for the next flush().

This kind of packetization can be useful in various scenarios, especially when the log is very large and you cannot send everything in one shot. The maximum size of a chunk can be set at run-time through *setSizeLimitPerChunk()*.

Please note that the flush() method guarantees that:

    * The data are sent in the order they were recorded 
    * During a single flush(), a record is sent at most once
    * At the first failure signalled by `flusher` callback, the flush() method ends

It returns true if flush succeeds (hence the log file is emptied), false otherwise.

### File systems

LoggerFS (the main class of the library) requires a file system to read and write the log. At the time of writing, there are few supported filesystems with minor differences between ESP32 and ESP8266:

* `SPIFFS`: this is the most popular filesystem for internal flash. On ESP32, you have to `#include <SPIFFS>`.
* `LittleFS`: this will be the replacement meant for SPIFFS since it has higher performance and directory support. You have to `#include <LittleFS.H>`.
* `SD` (only for ESP32): the standard file system for micro SD cards on ESP32. It supports both FAT and FAT32. You have to `#include <SD.h>`. Note that in ESP8266 core, there is an `SD` class, but since it doesn't derive from `FS`, it doesn't work with ESP Logger. Actually, it is just a wrapper over SDFS, only for compatibility with older code.
* `SDFS` (only for ESP8266): the standard file system for micro SD cards on ESP8266. It supports both FAT and FAT32. You have to `#include <SDFS.h>`.

Be sure that the log filepath exists before calling `begin()`. Some file systems create the necessary path when opening a file (like SPIFFS), other won't (like SD for esp32).

### Other APIs and examples

You may look at the *examples* folder for working sketches and you may look at the *commented* header files (*.h) for the details.

## Limitations

ESP Logger is designed to log human-readable data, this means that data should be composed only by readable characters (excluding also carriage return and newline).

Data corruption is not managed, hence there is no CRC or other integrity system to check that your log file is not altered. It assumes that the flash memory is reliable and no other code will access the log file.