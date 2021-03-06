# Porting from TXLib to TXLin

Usually, the process is straight-forward and the only thing is required is changing the header from TXLib.h to TXLin.h. However, there are still some quirks that you need to know if your application is a lot more complex than just printing "Hello world" on the screen.

This guide focuses on optimizing apps for TXLin while retaining their compatibility with TXLib.

## How do I include TXLib on Windows and TXLin on Linux/macOS in my code?
This is pretty straight-forward:
```
#ifdef _WIN32
#include "TXLib.h"
#else
#include "TXLin.h"
#endif
```

## Detecting TXLib from the source code
TXLib defines a special macro ``THIS_IS_TXLIN``. You can check for its presence using the ``#ifdef`` macro conditional. Example:
```
#ifdef THIS_IS_TXLIN
txSetConsoleAttr(TX_LIGHTGREEN);
printf("You use TXLin on Linux or macOS. I like both of these operating systems.\n");
#else
txSetConsoleAttr(TX_RED);
printf("You use TXLib on Windows. I prefer macOS over Windows.\n")
#endif
```

## Use txSticky() to keep the window opened
When you open a window in TXLib, it tends not to close until the user does so. However, this is not the case with TXLin. Due to SDL limitations, a function that imitates TXLib's "sticky window" functionality was added to TXLin. That function is ``txSticky()``. This function keeps the window open until the user clicks the close button or presses Command+Q (Alt+F4 on Linux) on his/her keyboard. If you do not add this function after you finish drawing everything on the drawing canvas, then you'll never see the resulting window and canvas.

Here's a simple Hello world program in TXLin:
```
#include "TXLin.h"

int main() {
	txCreateWindow(320, 480);
	txTextOut(6, 6, "Hello, TXLin!");
	return 0;
}
```

Now, if you compile and run this piece of code, you will see nothing happen on the screen. To actually see the window, we will add ``txSticky()`` after all our drawing operations, in this case, after ``txTextOut``:
```
#include "TXLin.h"
#ifndef THIS_IS_TXLIN
#define txSticky() (void)(nullptr) // to avoid miscompatibility of TXLin with TXLib
#endif

int main() {
	txCreateWindow(320, 480);
	txTextOut(6, 6, "Hello, TXLin!");
	txSticky();
	return 0;
}
```

Now you should see a window with the text "Hello, TXLin!" appear on your screen. It won't go away until you click the close button.

## Configurable and const static variables from TXLib are not available
So if you try change one of them in your program and then try to compile it with TXLin, you will get an error. To fix that, wrap around the code that modifies these variables in an ``#ifndef THIS_IS_TXLIN`` block. For example:
```
#ifndef THIS_IS_TXLIN
// change TXLib variables
_txLogName = "txlib_debuginfo.log";
#endif
```

## No deprecated API implemented
Since TXLin is a modern library, none of the deprecated TXLib API was ported. If you have TXLib code that uses these functions and you want to port it over to TXLin, then implement these functions manually.

These functions are:
- ``int random (int range)``
- ``double random (double left, double right)``
- ``bool In (Tx x, Ta a, Tb b)``
- ``bool In (std::nomeow_t, Tx x, Ta a, Tb b)``
- ``bool In (const POINT& pt, const RECT& rect)``
- ``bool In (const COORD& pt, const SMALL_RECT& rect)``
- ``std::nomeow_t`` is considered a deprecated type by TXLin and is not supported

## txSqr and txPI are now just macroses
txSqr function is actually a macros in TXLin. It points to a different define called ``TXLIN_UNPORTABLEDEF_SQUARE``. And yes, it won't troll the user anymore.

txPI is also a macros now and its value is just 3.14.

## TXLin's Monolithic Font
TXLin draws all the letters and characters by itself using its own built-in font. This was added so that the library would be more portable. However, the font has a fixed size of 6px and it cannot be changed on runtime. It can only be changed while linking the application. This limitation will be removed in a future release, however, for now you'll have to set the font width and height by yourself before the ``#include "TXLin.h"`` line if ypu was to change the font size. For example:

```
#define TXLIN_TEXTSET_WIDTH 12
#define TXLIN_TEXTSET_HEIGHT 12
#include "TXLin.h"

int main() {
	txCreateWindow(400, 400);
	txSetColor(TX_GREEN);
	txTextOut(60, 60, "This font is huge!");
	return 0;
}
```

Because TXLin provides its own font, other fonts cannot be used. This means that functions like txSelectFont will do nothing. Also, text alignment is not supported. It will be added to a future release.

## txInputBox returns a char*, not a const char*
This means that you'll have to free the memory allocated by the string returned by txInputBox by yourself.
```
char* favoriteColor = txInputBox("What is your favorite color?");
if (favoriteColor == nullptr)
	txMessageBox("I can't read the answer you've entered. Sorry. :-("); // your should not free a nullptr variable
else {
	txMessageBox(favoriteColor, "Your favorite color is...");
	free(favoriteColor); // free the memory allocated by the result string
}
```

## txHSL2RGB and txRGB2HSL are not available (txDialog is not available either)
There is actually a reason for this. Both of these functions, as well as the txDialog class, are really rarely used.

## txBitBlt does not work properly
txBitBlt currently does nothing, unfortunately. Unless I find a way to copy SDL_Renderer* to SDL_Renderer* , I won't be able to implement txBitBlt.

## Notice for C++98 compilers users
When building your program, right before the ``#include "TXLin.h"`` you'll have to add this line:
```
#define nullptr 0
```
This is needed because TXLin uses nullptr, not NULL, and, as such, most C++98 compilers (and there is no nullptr in C++98) fail to build the library. This is a workaround for these compilers.

## Some new features
### ``bool txSelectMouse(CURSORREF mouseId = TM_DEFAULT)``
Function that allows you to change system mouse pointer. 

``mouseId`` accepts these values:
- ``TM_DEFAULT`` - the default one
- ``TM_WAIT`` - "beachball" on macOS or hourglass on Linux
- ``TM_PROHIBITED`` - cross cursor
- ``TM_HAND`` - hand cursor
- ``TM_NONE`` - hide the cursor

Returns ``false`` if something goes wrong. Otherwise, returns ``true``.

Example:
```
txSelectMouse(TM_NONE);
txTextOut(7, 7, "Digital cat says: \"I ate your mouse. Want to bring it back? Then wait 6 seconds and you will get a new one!\"");
txSleep(6000);
txSelectMouse(TM_DEFAULT);
```

### ``char* txPassword()``
Function that shows a password dialog. It returns a ``char*``, so you'll have to free the memory by yourself.

Returns ``nullptr`` on failure. Otherwise, the password that the user has entered will be returned.

Example:
```
txMessageBox("To continue, enter your password.");
char* passwordString = txPassword();
if (passwordString == nullptr)
	txMessageBox("Error. Sorry for that.");
else if (strcmp(passwordString, "topsecretpasswordthatforsomereasonisstoredincode") == 0)
	txMessageBox("Access granted.");
else
	txMessageBox("Access is denied.");
free(passwordString);
```

### ``char* txTextDocument(const char* filename)``
Function that reads a text file to a string. It returns a ``char*``, so you'll have to free the memory by yourself.

Returns ``nullptr`` on failure. Otherwise, contents of the text document are returned.

Example:
```
txMessageBox("Gonna read the contents of \"poem.txt\".");
char* poemCtnt = txTextDocument("poem.txt");
if (poemCtnt == nullptr)
	txMessageBox("Oops, it does not exist. :-/");
else
	txMessageBox(poemCtnt);
free(poemCtnt);
```

### ``bool txRemoveDocument(const char* filename2)``
Function that removes a file.

Returns ``false`` if something goes wrong. Otherwise, returns ``true``.

Example:
```
txTextOut(7, 7, "Uh oh, your cat is hungry.");
txTextOut(7, 20, "Feed him with one of your files.");
char* path = txSelectDocument("Pick one of your mice", "*.mouse");
if (path == nullptr)
	txTextOut(7, 33, "Your cat is still starving and you didn't feed him anything.");
else if (txRemoveDocument(path) == false)
	txTextOut(7, 33, "Your cat does not have permission to eat that mouse.");
else
	txTextOut(7, 33, "Thank you!");
free(path);
```

### ``char* txSelectDocument(const char* text = "Please select a file to continue.", const char* filter = "*")``
Prompts a user to select a file matching filter ``filter``. It returns a ``char*``, so you'll have to free the memory by yourself.

Returns ``nullptr`` on failure. Otherwise, full path to the selected file is returned.

Example:
```
char* textFilePath = txSelectDocument("Choose a text file to read aloud.", "*.txt");
if (textFilePath == nullptr)
	TX_ERROR("textFilePath is nullptr.");
char* textReadFromFile = txTextDocument(textFilePath);
if (textReadFromFile == nullptr)
	TX_ERROR("Access is denied.");
txSpeak(textReadFromFile);
free(textReadFromFile);
free(textFilePath);
```

### ``const char* txCPUVendor()``
Function that returns the name of the company that made your CPU.

Returns "Unknown" on failure. Otherwise, returns one of these values:
- *Intel* if an Intel CPU was detected
- *AMD* if an AMD CPU was detected
- *VIA* if a Centaur, Cyrix or VIA notebook CPU was detected
- *Transmeta* if a Transmeta Crusoe CPU was detected
- *Parallels* if TXLin is running in a Parallels VM
- *VMware* if TXLin is running in a VMware VM
- *QEMU* if TXLin is running in a KVM or bhyve VM
- *Microsoft* if TXLin is running in a Hyper-V VM

Example:
```
if (txIsMacOS() && strcmp(txCPUVendor(), "AMD") == 0)
	txMessageBox("You use a Hackintosh. You cheater.");
else if (strcmp(txCPUVendor(), "Parallels") == 0 || strcmp(txCPUVendor(), "VMware") == 0)
	txMessageBox("You use macOS in a virtual machine.");
else
	txMessageBox("You might be using a real Mac (or you just have an Intel-based PC with Hackintosh).");
```

### ``bool txSpeak(const char* stringToSay)``
Function that detects a TTS engine on your computer and, using the default voice, says the string ``stringToSay``. This function is hightly unportable, as it uses std::system to gather information about the installed text-to-speech software. It also uses std::system to invoke that software.

Currently, these TTS engines are known to work:
- *Apple VoiceOver* on macOS
- *espeak* is known to work on Linux, *espeak-ng* should work too
- *festival* support is experimental, but it should work on Linux

It is not recommended that you put a string that is not English. You may try, but chances are that it will only work on macOS, as txSpeak does not do any language detection and only Apple's VoiceOver is able to handle different languages using a voice that might not even support that language.

Also, avoid punctuation like the exclamation mark or quotes. This is problematic for ``txSpeak`` to parse.

Returns ``true`` if everything goes well, otherwise, returns ``false``.

Example:
```
txMessageBox("You might be amazed by what you are about to hear.");
if (txSpeak("TXLin can speak. Yay."))
	txMessageBox("Should I ask Ilya Dedinsky to add this to TXLib? If yes, write me to <timprogrammer@rambler.ru>.");
else
	txMessageBox("This is not fair. You don't have TTS software installed. Maybe install espeak?");
```

__FUN FACT:__ macOS users have an exclusive opportunity to change the VoiceOver voice used by txSpeak. To do that, right before ``#include "TXLin.h"`` add this line:
```
#define TXLIN_MACOS_VOICEOVERVOICE "[your preferred voice]"
```
Obviously, you should replace "[your preferred voice]" with the voice that you want to use. If you are unsure about the voices installed on your Mac, open Terminal.app and run ``say -v ?`` to get a list of preinstalled voices.

### ``bool txIsLinux()``
Returns ``true`` if TXLin is running on Linux or ``false`` if it is running in either macOS or other UNIX-like OS.

### ``bool txIsMacOS()``
Returns ``true`` if TXLin is running on macOS or ``false`` if it is running in either Linux or other UNIX-like OS.

### ``bool txIsFreeBSD()``
Returns ``true`` if TXLin is running on FreeBSD or ``false`` if it is running in either macOS, Linux or other UNIX-like OS.

### ``txthread_t txSplitThread(void (*splitThreadFunc)(bool))``
Function that runs a callback thread function specified by the argument ``splitThreadFunc`` in a seperate thread and returns the variable representing this thread. 

The callback thread function must return ``void`` and accept only one argument - a ``bool`` variable. That ``bool`` variable is set to ``true`` if the experimental multithreading API is enabled and that function is running in a seperate thread. If that API is disable, the variable is set to ``false``. 

Why is this important? Due to some limitations, you cannot create any new windows, surfaces, message boxes or use flood fill in a new thread. From a new thread, you can only use this subset of API:
- ``txThreadLine`` (``txThreadLine_sepDC`` to specify the drawing context manually), ``txThreadSleep``
- color-changing API (``txSetColor``, etc)
- console API
- sounds and mice API


Example:
```
void threadMain(bool isRealThread) {
	txThreadSleep(1000);
	 if (isRealThread)
	 	txTextOut(6, 6, "Hello from the second thread!");
	 else
	 	TX_ERROR("Multithreading API is disabled.");
}

int main() {
	txCreateWindow(800, 600);
	txClear();
	txthread_t otherThread = txSplitThread(threadMain);
	while (txThreadRunning(otherThread))
		txLine(5, 5, rand() % 18, rand() % 18);
	txTextOut(12, 6, "Hello from the first (main) thread!");
	txSticky();
	return 0;
}
```

### ``bool txJoinThread(txthread_t thread)``
Function that freezes the current thread while thread ``thread`` is running. 

Returns ``true`` on success, ``false`` on fail.

### ``bool txThreadRunning(txthread_t thread)``
Returns ``true`` if thread ``thread`` is running. Otherwise, returns ``false``.

### ``bool txStopThread(txthread_t thread)``
Function that force stops thread ``thread``.

Returns ``true`` on success. Otherwise, returns ``false``.

### ``int txThreadSleep(unsigned int millisecs)``
Function that freezes the thread for ``millisecs`` milliseconds. Unlike ``txSleep``, works inside threads created by ``txSplitThread``.

Returns 0 on success, otherwise, -1 is returned.