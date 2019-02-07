---
layout: post
title: Flutter- Bottom Navigation Bar- Navigation using Dart Streams & BLoC Pattern<img src="/assets/images/flutter-blog/1_S4j5v4PwfukdxNutxB83BQ.gif" style="width:100%"/>
comments: False
categories:
- blog
---
---
Dart’s StreamController and Flutter’s StreamBuilder are powerful tools to achieve a better design and intra-app communication.
## Prerequisites
- Basic understanding of Flutter SDK and Scaffold Material Widget.
- Dart Programming Language. (Basic idea of Streams).
- BLoC Pattern — (Writing logic separate from UI in a Dart class and sharing it among Widgets).
### Main focus is Scaffold Widget’s body and bottomNavigationBar sections.
> "The Scaffold widget takes a number of different widgets as named arguments, each of which are placed in the Scaffold layout in the appropriate place."

The most significant among those named arguments are
- appBar: (The top section)
- body: (Main body section)
- bottomNavigationBar: (Bottom section)

Navigating between screens using Scaffold’s bottomNavigationBar can be achieved in many different ways. Most common way is

- Hold the current BottomNavigationBarItem index value inside the State of a StatefulWidget every time a BottomNavigationBarItem was tapped,
- and
- Call that State’s setState() to repaint the whole widget tree to reflect those changes.

A slightly better approach can be using a combination of

- BLoC — to separate business logic from UI
- Dart’s StreamController — to communicate the UI intents.
- Flutter’s StreamBuilder — to trigger UI updates.

## Advantages

- Using BLoC we can achieve clear separation of business logic from UI.
- No need to store current BottomNavigationBarItem index inside the State.
- No need to call setState() every time which re-creates the entire widget tree.
- Use StreamBuilder to re-create only a sub-tree or widget(s) instead of entire widget tree.

So, how can it be achieved?

## Two key things

- Communication between the Scaffold’s bottomNavigationBar and body sections is done using Dart’s StreamController.
- Use Flutter’s StreamBuilder to re-paint just the body section and the BottomNavigationBar.

### First thing we need a BLoC

Let’s create a BLoC. This BLoC holds

- an enum to represent various BottomNavigationBarItems.
- a dart StreamController object.
- a default Navigation Bar Item enum for initial state.
- a function which can be attached to onTap() gesture of BottomNavigationBar on UI side.
- a method to close the stream.

Lets go over each piece of BLoC in detail…

- an enum to represent various BottomNavigationBarItems.

<img src="/assets/images/flutter-blog/1_WUVV8K5nkXxAPPz5h9_B2Q.png" style="width:100%"/>

by listing out the Bottom NavBar Items as enum fields it’s easy to read the code and we can take advantage of enum indexes.

- A Dart StreamController.

<img src="/assets/images/flutter-blog/1_kMTdQoepMcxwppQzCpsPHw.png" style="width:100%"/>

Note: By default, Dart StreamController’s Stream is not a broadcast type stream. We can create one by calling broadcast() as shown in the screen print.

We need a broadcast type stream because, both body and bottomNavigationBar sections of the Scaffold listen-on the same stream to switch view based on current NavBarItem tapped.

- A default Navigation Bar Item enum for initial state

<img src="/assets/images/flutter-blog/1_RTHezw4t9fxpzKlNuDKfUw.png" style="width:100%"/>

Now, we need to set a default NavBarItem to show the default screen and NavBarItem selected. Let’s just set HOME as default.

- A function which can be attached to onTap() gesture on UI

<img src="/assets/images/flutter-blog/1_gheH3vcKHQQF_7hR4xOiCQ.png" style="width:100%"/>

This function holds the index tracking logic inside the BLoC separate from UI. This gets called out by onTap() when any BottomNavigationBarItem is tapped and receives the index of the tapped BottomNavigationBarItem.

- Finally a handy method to close the stream (It’s a good practice to close the streams when not needed)

<img src="/assets/images/flutter-blog/1_ejvhX_KDllYQ0YkXatqR0A.png" style="width:100%"/>

Because navBarController is a private field inside the BLoC we need a handy method to close the stream from outside of the BLoC.

### Now, UI part of the app

- A StatefulWidget
A StatefulWidget/State combo is not necessarily needed because we are not going to depend on setState() method anyway.

<img src="/assets/images/flutter-blog/1_wW7xLPKmi9L_pZYVDr3-cg.png" style="width:100%"/>

But, closing the Dart Stream when not needed is a best practice. This can be achieved by calling BLoC’s close() method inside State’s dispose() lifecycle method. So, we need a StatefulWidget here.

- A State

<img src="/assets/images/flutter-blog/1_eI2PmOUKZLN-G_Fqw3y3AA.png" style="width:100%"/>

State’s lifecycle methods initState() creates the BLoC and dispose() closes the stream when not needed.

- A StreamBuilder to switch the body section by listening to the NavBar item stream of the BLoC.

<img src="/assets/images/flutter-blog/1_T8VMzynTw_igVh0cfXtPfA.png" style="width:100%"/>

StreamBuilder in body section listens to BLoC’s stream and based on the NavBarItem enum in the current snapshot, it switches to corresponding view. Initial snapshot has the default NavBarItem (NavBarItem.HOME) and body will show homeArea().

- Another StreamBuilder to update the BottomNavigationBar to highlight the BarItem selected and feed the stream with onTap()

<img src="/assets/images/flutter-blog/1_QAeh3sHPURHVBRtFFSzclA.png" style="width:100%"/>

This StreamBuilder listens to same stream but builds the BottomNavigationBar and highlights the BottomNavigationBarItem based on the NavBarItem enum in current snapshot of the stream. So, initial snapshot has default NavBarItem (HOME) and Home BarItem gets highlighted.

onTap: Calls the BLoC’s pickItem() function and supplies selected BarItem’s index value on tap. BLoC’s pickItem() feeds-in the stream with a NavBarItem enum based on the index it receives. This triggers the whole action.

## In Action…

<img src="/assets/images/flutter-blog/1_S4j5v4PwfukdxNutxB83BQ.gif" style="width:100%"/>

Full source code is [here…](https://github.com/mbsambangi/flutter_navbar_bloc_streams_demo)

# End Note
I am sure there are many different ways of doing it. But, the above approach is certainly one clean way to separate the logic from UI. Dart’s StreamController and Flutter’s StreamBuilder are powerful tools to achieve a better design and intra-app communication. I hope this article is useful.
