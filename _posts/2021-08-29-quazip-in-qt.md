---
layout: post
toc: true
title: "Using QuaZip in a Qt Application"
categories: programming
tags: [qt, C++, QuaZip]
---


## 3rd Party Libraries for C++ Apps: Bane of Existence

It's about time that I brought my manga reading application back to life with my Qt skills I've acquired from work.

Extracting .zip and .cbz files are an integral part of the application.  The most luck I've had in finding a library that performs this functionality is QuaZip.  And luck is a bit of an overstatement.

I don't know exactly what I did almost three years ago when I managed to build it and successfully `#include` it inside of my project files.  I believe the method ended up being including all of the headers and source files from QuaZip into my application - a bloated and cumbersome way.  

This time around, I figured out how to build QuaZip and zlib and then simply add the shared object files to my Qt project.  Much cleaner, and it works perfectly.

Let's go over how I did this.  Hopefully it will help someone out there.

## Versioning

- **Operating System**: Ubuntu 20.04
- **Qt**: 5.12.8
- **zlib**: 1.1
- **QuaZip**: 0.7.3
- **CMake**: v.3.21.2

## Taking Care of zlib

QuaZip is dependent on zlib.  zlib is a compression library that is apparently included in Qt installations.

I searched for it extensively inside of the Qt installation, finding it in a 3rd party folder.  It seemed inadequate for the purposes of linking it with QuaZip, so I went the route of installing it from scratch with the following steps inside of my home directory.

```
wget http://www.zlib.net/zlib-1.2.11.tar.gz
tar -xvzf zlib-1.2.11.tar.gz
cd zlib-1.2.11/
./configure --prefix=/usr/local/zlib
sudo make install
```

This placed the shared object file inside of `/usr/local/zlib` since I specified a `prefix` in the `./configure` command.

## Taking Care of QuaZip

When going back to use QuaZip this time around, referencing [its page on the internet](http://quazip.sourceforge.net/), I did not realize that this documentation was out of date.  [Github](https://github.com/stachenov/quazip) shows that there is a newer version of QuaZip that uses **CMake**.

Instead of bothering with `qmake` again, I figured I would give this a try.

I downloaded the newest QuaZip release and then `cd`'d into the extracted directory.  Then I ran

```
cmake -S . -B ~/quazip -D QUAZIP_QT_MAJOR_VERSION=5
cmake --build ~/quazip
```

This built the necessary components into a folder of my choosing, in this case `~/quazip`.  I specified the Qt version to be 5 since I am running `5.12.8`, but you can also specify 4 or 6.

Then I installed it with the following command.

```
cmake --build ~/quazip --target install
```

It installed the .so file inside of `/usr/local/lib/` as `qauzip1-qt5.so` and put the included headers inside of `/usr/local/include/QuaZip-Qt5-1.1`. 

## Integrating QuaZip into the Qt Project

This was the most difficult part for me, as I was unfamiliar with how to dynamically link a 3rd party library to my Qt application.  I was initially looking for how to do this with QuaZip, but I abstracted away from this single use case and looked at how to do this in general so I could extrapolate.

The two 3rd party dynamic linking methods for Qt most commonly mentioned seemed to be:

- Importing literally all of the library files into a `dependencies/` folder in your project and then including the QuaZip `.pri` file in your `.pro` file
- Importing the shared object libraries that were generated to create a **dynamic link**

I did not want to go down the path of bloating my software with all of the QuaZip and zlib sources and headers, so I opted for the second option.

An easy way to do this was going into the `.pro` file, right clicking, selecting `Add library...`, selecting `External library`, selecting the `.so` file that was generated for QuaZip, and making sure the `Platform` was specified to only be `Linux`.  The `Include path` was automatically filled in for me when I selected to shared object file.  It generate the following in my `.pro` file.

```
unix:!macx: LIBS += -L$$PWD/../../../usr/local/lib/ -lqauzip1-qt5

INCLUDEPATH += $$PWD/../../../usr/local/include/QuaZip-Qt5-1.1
DEPENDPATH += $$PWD/../../../usr/local/include/QuaZip-Qt5-1.1
```

This is `dynamic linkage`.

I did the same thing for zlib, which sat in `/usr/local` alongside QuaZip.  The following was generated in my `.pro` file after adding the shared object library.

```
unix:!macx: LIBS += -L$$PWD/../../../usr/local/zlib/lib/ -lz

INCLUDEPATH += $$PWD/../../../usr/local/zlib/include
DEPENDPATH += $$PWD/../../../usr/local/zlib/include
```

It worked.

After I finish developing my application, the next step is going to be figuring out how to bundle these 3rd party libraries inside of the installations for others to use on their own machine.
