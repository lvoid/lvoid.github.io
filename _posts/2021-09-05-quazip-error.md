---
layout: post
toc: true
title: "Handling QuaZip mdUnzip Mode Error"
categories: programming
tags: [qt, C++, QuaZip]
---

## Background

My last post was all about building, installing, and importing the 3rd party library QuaZip into my Qt application.

Unfortunately, there are not very many 3rd party libraries for extracting and reading from zip files in-code for C++ Qt applications, and QuaZip was the best I found.  Despite the cryptic error I was getting, I was determined to make it work.

## The Problem

Right now, I have a basic function in my application that serves the purpose of extracting and reading from a .cbz file and then creating `QImage` objects from these to be placed into a `QList<QImage>` for reading on the front end.

```c++
bool Reader::loadCbzFile(QString cbzFile)
{
    qDebug() << cbzFile;
    
    QStringList allFiles = JlCompress::extractDir(cbzFile);

    QImage newImage;
    for(int file = 0; file < fileList.size(); file++) {
        QImageReader reader(allFiles[file]);
        newImage = reader.read();

        if(newImage.isNull()) return false;

        m_mangaImages.push_back(newImage); //save image to class variable
    }

    qDebug() << m_mangaImages.size(); // How many files did we extract?
    return true;
}
```

Before this function was called, a QML file chooser appeared to allow the user to select a file from their system, and this file URI was the QString that was passed to the function.

Each time this code was run, I received the following error:

```
QuaZip::getFileNameList/getFileInfoList(): ZIP is not open in mdUnzip mode
```

I did not know what this meant.  Digging through the source code of QuaZip, I found that it had an enum to specify the type of open() for a QuaZip object.  Since I was opening a zipped file, I specified the type as `QuaZip::mdUnzip`, yet I was getting an error saying that `ZIP is not open in mdUnzip mode`.  

This was a contradiction and did not make sense since I had indeed specified it to be mdUnzip, and delving further into the source code for where this error was generated was not very enlightening - it was literally just checking to see if the QuaZip object was in mdUnzip mode.

I became fixated on the fact that it must have been an error on my behalf when it came to building QuaZip or zlib.  This was the part I was most unfamiliar with after all, and I have gone through hell in the past trying to build older versions of QuaZip for different operating system.

I rebuilt and reimported zlib and QuaZip multiple times using different methods, each of which yielded the same result.

I even opened an issue on the QuaZip Github and the developer did not know what to make of it.

## The Solution

In the function, I am printing out the path of the file in a debug statement.  From rudimentary glances each time the program ran, it looked correct.

What would happen if I manually put the file URI into the function?

I did that, and it worked.  There were no error printouts, and it showed the size of the populated list to be in the 200's, as it should've been.

Why was it doing this?

I took a closer look at the URI printed from the file chooser and the URI I manually input.

The file chooser URI passed from QML was literally missing a **/** at the beginning.  

That was all.  

Since it was missing this **/** for the absolute path, it was not actually finding the .cbz file to extract.  Instead of stating that it could not find the file, however, it printed out a cryptic, unrelated error.

I fixed this by simply appending a "/" to the beginning of my URI.

## The Lesson

Do not rely on a 3rd party application for anything other than its intended functionality.  There were not adequate error checks in QuaZip to handle a file not found issue, so it instead gave me a confusing error that sent me down a rabbit hole of rebuilding and reimporting libraries.

Whether or not a file exists or a path is correct is my responsibility to check in the code.  

Furthermore, this debacle reinforced something I have suffered from many times in my career: when you look at a URI, make sure to study every single character painstakingly to ensure it is correct.

My code now looks like this, with `openFiles(QStringList fileList)` the slot that is first called from QML.

```c++
void Reader::openFiles(QStringList fileList)
{
    for(int fileIndex = 0; fileIndex < fileList.count(); fileIndex++) {
        QString currentFilePath = fileList.at(fileIndex);

        if (currentFilePath.endsWith(".cbz") || currentFilePath.endsWith(".zip")) {
            /* If the user selected a .zip or .cbz, select only the first file */
            if(fileExists(currentFilePath)) {
                extractCbzFile(currentFilePath);
            }
            else {
                /* Possible URI is missing a "/", append one and check if it works */
                QString ammendedFilePath = "/" + currentFilePath;

                if(fileExists(ammendedFilePath)) {
                    extractCbzFile(ammendedFilePath);
                }
                else {
                    qDebug() << "ERROR: File not found.  Check URI:" << ammendedFilePath;
                }
            }
        }
        else if (currentFilePath.endsWith(".png") || currentFilePath.endsWith(".jpg")
                 || currentFilePath.endsWith(".jpeg")) {
            /* TODO: If the user selected a(n) image, send the full list */

        }

    }
}

bool Reader::fileExists(QString path) {
    QFileInfo check_file(path);

    // check if file exists and if yes: Is it really a file and no directory?
    if (check_file.exists() && check_file.isFile()) {
        return true;
    } else {
        return false;
    }
}

bool Reader::extractCbzFile(QString cbzFile)
{   

    QStringList allFiles = JlCompress::extractDir(cbzFile);

    QImage newImage;
    for(int file = 0; file < allFiles.size(); file++) {
        QImageReader reader(allFiles[file]);
        newImage = reader.read();

        if(newImage.isNull()) return false;

        m_mangaImages.push_back(newImage); //save image to class variable
    }

    qDebug() << m_mangaImages.size();
    return true;
}

```
