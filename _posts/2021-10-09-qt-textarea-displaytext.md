---
layout: post
toc: true
title: "QML TextArea's Multiple Text Fields"
categories: programming
tags: [qt, qml]
---


## Versioning

- Qt 5.12.8
- QtQuick 2.0
- QtQuick.Controls 2.5

## A Puzzling Story

Although Qt debatably provides solid documentation (finding a specific version is a hassle), debugging QML in QtCreator is not the most enlightening experience.  In fact, I often have to create my own system of debugging using `console.log()` and observing the output of a log file when the program is running.  

I don't think that this is something a programmers should have to do when utilizing a modern framework or language, but I can deal with it.  I've come to embrace Qt and its faults after doing multiple year-plus long projects at work using it.

QML, the declarative frontend language for QtQuick bundled with Qt, has some very practical widgets, including `TextField` for user input.

I had to customize the `TextField` extensively to best suit the needs of my application, including using the `inputMask` field of the widget.

This worked great magnificently in the UI.  It did exactly what I thought it would do.  It also manipulated its `text` field in a way that I did not expect, and I could not figure out how to derive the text I wanted to send to the backend until I took a look at Qt's documentation.

## Qt TextField Use Case

To provide some background, the project I'm working on requires a text fields that allows the user to input numbers in the format `mmm:ss`.

This numerical time value defines an interval which is displayed on a timeline.  For a given interval, there is a `startTime` and `endTime`, both being numerical fields that utilize a QML `TextField`.

## Input Validation

I went ahead and added the following values to the `TextField`:

```
placeholderText: "000:00"
inputMethodHints: Qt.ImhDigitsOnly
validator: RegExpValidator { regExp: /^([0-9\s]?[0-9\s]?[0-9\s]):([0-5\s][0-9\s])$ / }
```

This caused the `TextField` to display a default greyed-out text of `000:00` and restricted the user to inputting only certain values.  For example, the regular expression implemented makes it so the field can only receive a maximum time of `999:59`.

##  Explaining a hard-to-explain feature  

I knew what I wanted next, but I didn't know what it was called.

Whenever I was pressing backspace in the textfield, it was deleting all of the zeros.  I wanted the `TextField` to behave in a way that "backspace" would navigate back, but instead of deleting the zero, it would leave it in place and only navigate back to the previous digit.

I had seen this implemented in input fields in various applications, and I always thought it was convenient.

After some internet searches, I had discovered the answer was an `inputMask`.

I added the following to my `TextField` declaration:

```
inputMask: "000:00;0"
```

This added the behavior described above, creating a more efficient and tolerable interaction with the text fields.

All was well until I tried converting the text field contents to seconds.

## Conversion Algorithm

I made a little utility function in Javascript inside of my QML file to translate the text input in the `TextField` to an integer representing total seconds.  Here's what it looks like.

```
// mmm:ss -> (int) seconds
function transformTimeToSeconds(timeEntered) {
    var splitTime = timeEntered.split(":")

    var minutes = splitTime[0]
    var seconds = splitTime[1]

    if(minutes.charAt(0) === '0') {
        if(minutes.charAt(1) === '0') {
            minutes = minutes.charAt(2)
        }
        else {
            minutes = minutes.charAt(1) + minutes.charAt(2)
        }
    }


    var totalSeconds = (parseInt(minutes) * 60) + parseInt(seconds)

    return totalSeconds
}
```

It was working flawlessly before I introduced the `inputMask`.

The time slots that were now being generated were broken, and my log file would not tell me why.

I logged the output of the function above.  It produced `NaN`.  What sort of value was being sent to this function?

I logged `timeEntered` when the I entered `000:05` on the UI `TextField`.  It's output: `:5`

Hang on.  I had sent it the `text` member of the `TextField` component.  Why did the `text` not correspond to the text that that was clearly displayed on the UI?

It had something to do with the `inputMask`.  The value that was sent, although it was showing 0's on the front end, had stripped the zeros on the backend value.

What was I going to do?  This was the `text` member, right?  I thought about either:

- Creating a function to take this text and transform it, replacing zeros, and sending it to the `transformTimeSeconds()` above
- Modify the `transformTimeSeconds()` to account for these stripped values, adding several more boolean statement trees.

Things could get complicated with the above options.  Surely, there had to be another way.  did I have to get rid of `inputMask`, which was working so well?

My gut told me to look at the documentation.  What if there were multiple text fields?  I wouldn't put it past Qt.

## displayText and text

Sure enough, there it was.

In addition to the `text` member, which explicitly states it is the "...property that is the text in the TextField", there is a `displayText` member.

The description of `displayText`?

Pretty much the same.

Maybe this was it.  I did a some logging to see what value was sent when calling `TextField.displayText` instead.  The result?  `000:05`.

My application worked again.

## Googling is not always the answer

It's healthy to think for yourself.  Google is only so helpful.  After quite a few searches, I found no one else with this problem in Stack Overflow and various other forums.

This was something I had to solve on my own.  Good.  Trying to sift through endless webpages for an answer tends to break my flow, and doesn't make the problem solving nearly as fun.

It was a simple as using intuition and referencing the documentation.

Hopefully, this post can help someone who happens to run into the same esoteric issue.  Solving problems without any searching around can be rewarding, but there is also something to be said about saving time.
