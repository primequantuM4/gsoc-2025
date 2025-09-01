# Dart Terminal UI (TUI) framework 
# Introduction

This project took place as part of **Google Summer of Code (GSoC) 2025** with the Dart organization.  
The goal was to build a **terminal user interface (TUI) framework for Dart**, modeled after Flutter’s declarative API style.  

## Motivation

Every programming language that grows a healthy ecosystem eventually develops frameworks for building terminal-based applications. TUIs power developer tools, monitoring dashboards, setup wizards, and interactive apps that don’t need the overhead of a full GUI.  

Before this project, Dart had **no mature option** for creating TUIs. Developers who wanted to build an interactive terminal app were forced to reach for Rust, Go, or Python. That barrier discourages adoption because it interrupts the workflow: a Flutter developer shouldn’t have to switch languages just to build a companion CLI or TUI tool.  

Providing a Dart-native TUI framework fills this gap. It:  
- **Keeps developers inside the Dart ecosystem** instead of forcing them to context-switch into another language.  
- **Supports productivity** by letting teams reuse patterns they already know from Flutter, like declarative UI and state management.  
- **Expands Dart’s reach** into a new domain (CLI/TUI tooling), making it more appealing as a general-purpose language rather than being perceived as “just for Flutter GUIs.”  

The framework we built, **Pixel Prompt**, is designed to feel natural for any developer already comfortable with Flutter, but also approachable for newcomers. It lowers the barrier to writing interactive terminal apps in Dart, strengthens the language’s ecosystem, and tries to make Dart more competitive with languages that already have strong TUI support. 

# Background

## Why TUIs Matter

Terminal user interfaces (TUIs) remain an important part of modern software, even with the prevalence of graphical interfaces. They are used to build developer tools, monitoring dashboards, and interactive applications that don’t require the overhead of a full graphical environment. TUIs offer a middle ground: more user-friendly and interactive than plain command-line scripts, but far lighter than full GUIs.  

For developers, TUIs provide an ergonomic way to deliver functionality in constrained or server-side environments where graphical frameworks are impractical.

## Why Dart is a Good Fit

Dart is best known as the language behind Flutter, which popularized a declarative, widget-driven style of building user interfaces. That same design philosophy makes Dart a natural candidate for TUIs. Developers already familiar with Flutter can reuse concepts like components, state, and reactive updates in the terminal.  

While Dart is not as low-level as Rust or as minimal to write as Python, it strikes a useful balance: it is efficient enough to handle rendering and input for TUIs, while remaining accessible and easy to maintain. Importantly, a TUI framework also keeps Dart developers inside their ecosystem. A Flutter team that wants to build a command-line companion tool can now do so in Dart without context-switching into another language.  

## Context for Non-Specialists

Other ecosystems have already proven the value of strong TUI libraries. Python has **curses** and newer frameworks like **Textual**, Go has **Bubble Tea**, and Rust has **ratatui**. Each of these helped their language move beyond one domain and into another. Pixel Prompt plays the same role for Dart: it fills a missing gap, extends the ecosystem, and strengthens Dart’s positioning as a general-purpose language instead of being seen as “just for Flutter GUIs.”  

# Architecture & Design

When I started building Pixel Prompt, the most important design question was how to support multiple interactive components on the screen at once. In my earliest prototype, there was no clear way to manage focus or navigate between components, which quickly made the framework unusable. Solving that issue required rethinking how components were structured in the first place.  

## Component Model

The solution was to adapt ideas from Flutter’s architecture. Flutter uses a separation of concerns across **Widget → Element → RenderObject**. Pixel Prompt follows a simplified but conceptually similar approach with **Component → ComponentInstance**.  

- A `Component` is a lightweight, declarative description of what should appear on the screen (similar to a Flutter widget).  
- A `ComponentInstance` is the live object created from a `Component`, responsible for holding state, layout bounds, and rendering logic.  

This split made it possible to support const-like components for performance, while still keeping layout and rendering flexible.  

Architecture diagram can also be found at [exaclidraw](https://excalidraw.com/#json=PqN7x9JSuPEXttYLiWFwh,JDhDOS7R6yVrFcroKHr0kA)
<img width="2720" height="653" alt="image" src="https://github.com/user-attachments/assets/48601ad9-3717-41ca-9b8a-2e6d7958a779" />

A minimal example looks like this:  

```dart
class Counter extends Component {
  final int count;

  const Counter(this.count);

  @override
  ComponentInstance createInstance() => _CounterInstance();
}

class _CounterInstance extends ComponentInstance<Counter> {
  @override
  void render(CanvasBuffer buffer, Rect bounds) {
    buffer.drawAt(0, 0, "Count: ${component.count}", TextComponentStyle());
  }
  // ... other override elements like fitWidth(), fithHeight()...
}
```
Here, `Counter` is a `Component` that describes the UI, while `_CounterInstance` handles the rendering. This mirrors Flutter’s widget/element separation, but tuned for terminal rendering.
## Rendering System

Rendering in the terminal has its own challenges. Writing full screen buffers to `stdout` every frame is wasteful and makes testing harder. To address this, I implemented **double buffering** combined with **ANSI diffing**: instead of redrawing the entire screen, Pixel Prompt only writes the minimal set of changes needed to update the terminal.  

This optimization significantly reduced the output size (from ~110 KB down to ~20 KB in one test case) and simplified rendering performance. However, it also meant my original golden tests were no longer valid, since they expected full frame snapshots. The fix was to build a custom virtual terminal interpreter that could reconstruct the screen state from the diffs and verify correctness against that.  

## Input and Event Handling

Input is managed through a global event system that captures keyboard (and in some cases mouse) events from `stdin`. Components can subscribe to input events and update their state in response. This design allows building interactive UIs with familiar concepts like `setState`, mirroring the Flutter development experience.  

For example, a text input field listens for key presses and updates its buffer, while a game component like Snake reacts to arrow keys. These inputs feed directly into the component tree, triggering re-renders only where necessary.  

# Features Implemented

Pixel Prompt’s goal was to provide a complete and ergonomic framework for building terminal UIs in Dart. The following are the key features implemented:

## State Management

The framework supports a `setState`-style model similar to Flutter. Components can hold internal state, and calling `setState` triggers a re-render of only the affected components. This allows developers to build dynamic UIs without manually managing redraws or tracking which parts of the screen need updating.

## Component System

Pixel Prompt provides a flexible component system where developers can define reusable UI elements. Components are declarative, and the separation between `Component` and `ComponentInstance` ensures efficient rendering and layout calculations. Developers can extend either `BuildableComponent` or `StatefulComponent` and define their UI in the `build` method they provide. 

```dart
// State independent class
class TextHeader extends BuildableComponent {
  @override
  List<Component> build() {
    return [
      Row(children: [
        TextComponent('Text', style: TextComponentStyle(
              color: ColorRGB(122, 199, 246),
              bgColor: ColorRGB(13, 15, 12),
              padding: EdgeInsets.symmetric(horizontal: 2, vertical: 1),
            ),
        ),
        TextComponent(
            "Header",
            style: TextComponentStyle(
              color: ColorRGB(245, 147, 137),
              bgColor: ColorRGB(20, 9, 18),
              padding: EdgeInsets.symmetric(horizontal: 4, vertical: 1),
            ),
          ),
      ])
    ];
  }
}

// State dependent class
class CounterComponent extends StatefulComponent {
  int counter = 0;
  @override
  ComponentState<CounterComponent> createState() => CounterState();
}

class CounterState extends ComponentState<CounterComponent> {
  @override
  List<Component> build() {
    return [
      Row(
        children: [
          ButtonComponent(
            label: 'Increment',
            borderStyle: BorderStyle.rounded,
            outerBorderColor: ColorRGB(0, 200, 0),
            buttonColor: ColorRGB(0, 200, 0),
            onPressed: () {
              setState(() {
                component.counter++;
              });
            },
          ),
          ButtonComponent(
            label: 'Decrement',
            outerBorderColor: ColorRGB(255, 0, 0),
            buttonColor: ColorRGB(255, 0, 0),
            borderStyle: BorderStyle.rounded,
            onPressed: () {
              setState(() {
                component.counter--;
              });
            },
          ),
        ],
      ),
      TextComponent(
        'Counter: ${component.counter}',
        style: TextComponentStyle(
          color: ColorRGB(200, 200, 200),
          bgColor: ColorRGB(20, 20, 20),
          padding: EdgeInsets.symmetric(horizontal: 6, vertical: 1),
        ),
      ),
    ];
  }
}

// entry point
void main() {
  App(children: [
    TextHeader(),
    CounterComponent()
  ]).run();
}
```

<img width="305" height="178" alt="image" src="https://github.com/user-attachments/assets/424767aa-ee83-49bb-b3e7-4dfe9d857ea2" />

## Input Handling

Pixel Prompt implements a global event system for keyboard and mouse input. Components subscribe to relevant events and can react in real time. This enables interactive applications like games or forms.  

## Rendering Optimizations

Double buffering and ANSI diffing ensure that the framework only writes the minimal necessary changes to the terminal, reducing output size and improving performance. This optimization also makes golden testing feasible with a virtual terminal interpreter.

## Example Applications

To validate framework completeness, I built a few demo apps:  

- **Snake Game** – Demonstrates dynamic input handling, state updates, and screen re-rendering.
![snake_demo](https://github.com/user-attachments/assets/3cdb67f7-00e5-4211-a79d-56be819d8385)

- **Todo App** – Illustrates form inputs, focus management between components, and state persistence.  
  <img width="343" height="290" alt="image" src="https://github.com/user-attachments/assets/109d6397-312e-4d23-9ec8-fdd35b1a78d6" />


- **Stopwatch** – Highlights real-time updating of components with timed state changes.  
  <img width="344" height="144" alt="image" src="https://github.com/user-attachments/assets/cc481e14-32e7-4bbe-835c-b64abe5ec195" />

## Challenges Faced

Building a TUI framework from scratch in Dart surfaced several challenges, both from the ecosystem’s limitations and from the inherent quirks of terminal programming.  

### Windows Unicode Support

One of the earliest issues was handling Unicode rendering on Windows. While Linux and macOS terminals handle wide characters (like emojis or CJK scripts) consistently, Windows required additional work with [`ffi`](https://pub.dev/packages/ffi) and [`win32`](https://pub.dev/packages/win32) to properly support wide character output. Without this fix, text would appear misaligned or broken.  
- Related discussion: [Dart SDK issue #48329](https://github.com/dart-lang/sdk/issues/48329)

### Buffer Optimization and Testing

We wanted reliable golden tests to ensure rendering correctness.  
However, with **double buffering and ANSI diffing**, stdout only contained incremental updates (diffs) rather than full frames. This made traditional snapshot testing ineffective, since snapshots alone could not reconstruct the final screen state.

To solve this, we built our very own **custom virtual terminal interpreter**! that replays the diffs to rebuild the full screen. Golden tests then compare the reconstructed terminal state against the expected output, ensuring both correctness and performance remain verifiable.

### Input Latency
Another persistent issue is input latency at startup. Applications built with the framework sometimes take noticeable time before responding to keyboard input. My current investigation points to how stdin is initialized and consumed in Dart.  

Interestingly, when compiling applications with `dart compile`, the startup lag was virtually eliminated. This indicates that the latency is more a characteristic of running in the Dart VM rather than a limitation of the framework itself. While the input system works reliably in both modes, improving responsiveness in VM execution remains an open area for future exploration. ([Issue#44](https://github.com/primequantuM4/pixel_prompt/issues/44))

## Limitations  
Pixel Prompt already enables building interactive terminal applications, but with only three months for GSoC, there were many ideas and improvements that could not be implemented. The framework focuses on core primitives, leaving several clear areas for future development. All of these have corresponding GitHub issues so that contributors can pick them up.

One area that remains under development is support for **complex components**. While text rendering, checkboxes, and basic input fields are fully functional, larger interactive elements such as a `TextArea` for multiline editing or `Table` components for structured data are not yet implemented. These components require more sophisticated layout logic and careful handling of cursor movement. ([Issue#41](https://github.com/primequantuM4/pixel_prompt/issues/41))

The **layout engine** currently supports `PositionType.absolute` and `PositionType.relative`, but does not yet provide a full flex-like system capable of robust nested layouts. Creating such a system would require a significant expansion of the framework’s architecture. ([Issue#34](https://github.com/primequantuM4/pixel_prompt/issues/34))

A dedicated **animation system** is also missing. At present, animations are simulated by repeatedly triggering `setState`, which works but is neither efficient nor ergonomic. Implementing a declarative animation API would allow developers to specify timed transitions and effects more naturally. ([Issue#42](https://github.com/primequantuM4/pixel_prompt/issues/42))

Finally, there is no **visual debugger**. Developers currently cannot inspect the component tree or view layout decisions at runtime, which makes debugging more challenging. A tool similar to Chrome DevTools or the [Clay](https://github.com/nicbarker/clay) C UI library would allow developers to inspect component hierarchies and properties in real time. ([Issue#43](https://github.com/primequantuM4/pixel_prompt/issues/43))

**In summary, the current limitations are:**  

- Missing complex components (`TextArea`, `Tables`)  
- Lack of a flex-like layout system  
- No dedicated animation API (only manual frame updates)  
- No visual debugger for inspecting component trees  


## Future Work  

Looking ahead, there are several areas where Pixel Prompt can grow into a more complete framework. Many of these directions are natural extensions of the limitations discussed above.  

The most immediate need is a proper **animation API**. Instead of developers manually calling `setState` to fake animations, the framework should expose a dedicated system for timed transitions, tweens, and frame-based effects. This would make it much easier to create smooth loading indicators, dynamic UI transitions, or even simple games.  

Another significant step would be building a **visual debugger**. Much like Flutter’s DevTools or the Clay C UI library, the idea would be to let developers toggle an inspector view that displays the component hierarchy and layout details in real time. This would dramatically improve the debugging experience and lower the entry barrier for contributors trying to understand the internals.  

Expanding the **component library** is also a key priority. Implementing more advanced widgets such as `TextArea`, `Table`, `ListView`, and flexible layout managers would make Pixel Prompt usable for more realistic applications. This ties closely to the eventual goal of introducing a **robust layout system**. A flex-like or grid-based layout engine would give developers fine-grained control over positioning and nesting, without forcing them to micromanage bounds manually.  

Beyond technical features, the project will also benefit from clearer **contributor guidelines** and documentation. While the code is modular and tested, lowering the barrier to entry for new contributors is essential if Pixel Prompt is to evolve beyond a single-person effort. 

## Links  

- **Repository**  
  - [Pixel Prompt on GitHub](https://github.com/primequantuM4/pixel_prompt)
  - [Pixel Prompt on pub.dev](https://pub.dev/packages/pixel_prompt)

- **Example Applications**  
  - [Snake Game Demo](https://github.com/primequantuM4/pixel_prompt/blob/main/example/snake/snake_game.dart)  
  - [Todo App Demo](https://github.com/primequantuM4/pixel_prompt/blob/main/example/stateful_component_demo/bin/todo_demo.dart)  
  - [Stopwatch Demo](https://github.com/primequantuM4/pixel_prompt/blob/main/example/buildable_component_demo/bin/stop_watch_demo.dart)  

- **Relevant Dart/Flutter Resources**  
  - [Dart SDK issue #48329: Windows Unicode support](https://github.com/dart-lang/sdk/issues/48329)  
  - [ffi package](https://pub.dev/packages/ffi)  
  - [win32 package](https://pub.dev/packages/win32)  

- **Inspiration / References**  
  - [Clay (C UI Library)](https://github.com/nicbarker/clay)
  - [Flutter docs](https://api.flutter.dev)  
  - [Bubble Tea (Go TUI framework)](https://github.com/charmbracelet/bubbletea)  
  - [ratatui (Rust TUI framework)](https://github.com/ratatui-org/ratatui)  
  - [Textual (Python TUI framework)](https://github.com/Textualize/textual)  



