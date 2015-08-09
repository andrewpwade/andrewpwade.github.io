---
title: Compiling Zeal on OS X Yosemite
layout: post
excerpt: ""
---

Make sure Xcode is installed (the full package, not just the command line tools). This can be installed from the App Store.

Next, assuming homebrew is already installed:

~~~
brew install homebrew/versions/qt52
brew install libarchive
git clone https://github.com/zealdocs/zeal
cd zeal
~~~

Patch `src/src.pro`. Add the following lines near the top of the file:

~~~
INCLUDEPATH += /usr/local/opt/libarchive/include
INCLUDEAPTH += /usr/local/opt/libarchive/lib/
LIBS += -L/usr/local/opt/libarchive/lib -larchive
~~~

Now run the following (it ensures proper clean-up between compilation attempts):

~~~
( cd src && make clean ) ; make clean ; rm -rf bin ; /usr/local/Cellar/qt52/5.2.1/bin/qmake && make
~~~

Open the application:

~~~
open bin/Zeal.app
~~~

If Zeal opens successfully optionally create a DMG:

~~~
/usr/local/Cellar/qt52/5.2.1/bin/macdeployqt bin/Zeal.app -dmg
~~~
