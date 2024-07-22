# Embedded Swift Package on ESP-IDF

This is a template project to develop Embedded Swift Packages for the ESP-IDF platform.

## Installation

### Requirements

- Swift 6.0+ toolchain (currently only available as snapshot)
  - Install [swiftly](https://swiftlang.github.io/swiftly/)
  - Run `swiftly install -u main-snapshot`
  - _The project was confirmed working with `main-snapshot-2024-07-15` (check with `swiftly use`)_
- Recent installation of [ESP-IDF](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/get-started/index.html#installation)
  - Make sure you have access to `idf.py` from the command line ([Guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/get-started/linux-macos-setup.html#step-4-set-up-the-environment-variables))
  - _The project was confirmed working with `ESP-IDF v5.2.2-dirty` (check with `idf.py --version`)_
- A microcontroller with RISC-V architecture supported by ESP-IDF
  - _The project was confirmed working with `esp32c6`_
- _The project was confirmed working with Linux_

### Get started

1. Download or clone this repository
2. Open a console in the repository folder
3. Configure the Project
   - Run `idf.py set-target <device>`. Valid values for `<device>` can be listed with `idf.py --list-targets`. Do note that only RISC-V targets are supported by the swift compiler (and therefore this project)
4. Build the Project
   - Run `idf.py build`. This will also set up everything needed by sourcekit-lsp
5. Open `main/` in your preferred code editor. This is where your Swift Package lives
6. Well... Code, i guess?
7. Build the project: `idf.py build`
8. Upload and monitor the project: `idf.py flash monitor`


## LSP Support

If you use VSCode and the [Swift](https://marketplace.visualstudio.com/items?itemName=sswg.swift-lang) Extension, everything should work out of the box.

If LSP does not find a module, try rebuilding your project via `idf.py build`. If you have imported a Package or created a target with component requirements, make sure they are included in the `idf_component.yml` and run `idf.py reconfigure`, followed by `idf.py build`.

If not, some configuration of sourcekit-lsp is necessary to use the correct target and point it to the correct header files. Additional flags you need to provide to sourcekit-lsp:
- `-Xswiftc` `--target=riscv32-none-none-eabi`
- `-Xswiftc` `-enable-experimental-feature` `-Xswiftc` `Embedded`
- `-Xswiftc` `-wmo` : Required by the above
- `-Xcc` `-D__riscv` : Needed for some header files to resolve correctly
- `-Xcc` `--config=./.build/release/compile_flags.txt` : A clang configuration file holding all include directories of ESP-IDF, created during `idf.py build`

## Interfacing ESP-IDF

To use IDF libraries, create a new Target in your Swift Package, containing a header file which includes the IDF headers you want.

Example: Getting Access to FreeRTOS

- Add new Target to `Package.swift` and adding it as a dependency to all Targets that want to use it.
  ```swift
  let package = Package(
  [...]
    targets: [
        // 1. Add Target
        .target(name: "CFreeRTOS"),

        .target(
            name: "Main",
            // 2. Add Dependency
            dependencies: ["CFreeRTOS",]),
    ]
  [...]
  ```
- Add target sources. Add a folder to the `Source`-directory named after the new target. Add a `include` directory inside that. Add a C header file inside that. Your Source Directory should now look like this:
  ```
  Source
  +- CFreeRTOS
  |  +- include
  |     +- freertos_includes.h
  +- Main
     +- esp_main.swift
  ```
- Include all headers you need in the `freertos_includes.h`:
  ```c
  #include "freertos/FreeRTOS.h"
  #include "freeRTOS/task.h"
  ```
- Import the target in your swift source file. `esp_main.swift`:
    ```swift
    // 1. Import target
    import CFreeRTOS

    @_cdecl("app_main")
    func app_main() {
        print("🏎️+📦   Hello from an Embedded Swift Package")

        // 2. Use some C functions
        vTaskDelay(200)
    }
    ```

Do note that some constants are built using C macros. These can not be used directly in Swift. Instead, you need to create a C constant inside your header file for Swift to use. Fortunately, if this happens, the Swift Compiler lets you know gently, that it is unable to use C macros in Swift code.

To build upon the example above, `vTaskDelay` takes the amount of ticks to delay, but I want to declare the amount of milliseconds to delay.

- There is a C Macro constant for converting one to the other:
    ```c
    ticks = milliseconds / portTICK_PERIOD_MS;
    vTaskDelay(ticks)
    ```

- Assigning the macro value to a constant in `freertos_includes.h`:
    ```c
    #include "freertos/FreeRTOS.h"
    #include "freeRTOS/task.h"

    const TickType_t freeRTOS_Tick_Period = portTICK_PERIOD_MS;
    ```

- Makes it accessible in our Swift Module `esp_main.swift`:
    ```swift
    import CFreeRTOS

    @_cdecl("app_main")
    func app_main() {
        print("🏎️+📦   Hello from an Embedded Swift Package")

        vTaskDelay(200 / freeRTOS_Tick_Period) // waits for 200ms
    }
    ```

## Using ESP Components

If you want to use components from the [ESP component Registry](https://components.espressif.com/), all you have to do is declare them in a `idf_component.yml` in your Package root (so `main/idf_component.yml`). Instructions on how to populate this file can be found [here](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/tools/idf-component-manager.html).

Every time you change `idf_component.yml`, you need to run `idf.py reconfigure` in the project folder.

Using the component inside swift is similar to using built-in components. Please refer to [Interfacing ESP-IDF](#interfacing-esp-idf)

### Using Swift Packages that include ESP components

If you include a Swift Package that makes use of a non-builtin ESP Component, you must include its dependencies in your `idf_component.yml` and run `idf.py reconfigure` afterwards. A recompilation via `idf.py build` might also be necessary to regenerate LSP files.

Example: Including the Swift-Package [swift-led-strip](https://github.com/anders0nmat/swift-led-strip) that provides a Swift-Wrapper for [espressif/led_strip](https://components.espressif.com/components/espressif/led_strip)

`Package.swift`:
```swift
let package = Package(
    [...]

    dependencies: [
        .package(url: "https://github.com/anders0nmat/swift-led-strip.git", branch: "main"),
    ],
    targets: [
        .target(name: "CFreeRTOS"),
        .target(
            name: "Main",
            dependencies: [
                "CFreeRTOS",
                .product(name: "LedStrip", package: "swift-led-strip"),
        ]),

    [...]
```

`idf_component.yml`:
```yaml
dependencies:
  espressif/led_strip: "^2.4.1"
```

`esp_main.swift`:
```swift
import CFreeRTOS
import LedStrip

@_cdecl("app_main")
func app_main() {
    let strip = LedStrip(pin: 8, ledCount: 1)

    var is_on = false
    while true {
        is_on.toggle()
        switch is_on {
        case true:
            strip.setPixel(at: 0, to: Color(
                red: (0..<64).randomElement()!,
                green: (0..<64).randomElement()!,
                blue: (0..<64).randomElement()!))
        case false:
            strip.clear()
        }
        strip.refresh()

        vTaskDelay(500 / freeRTOS_Tick_Period)
    }
}
```


## How does it work?

It is a multi-step process, involving many "optional" steps to get sourcekit-lsp to work, but basically:

1. Do the usual ESP-IDF build setup, finding includes, resolving components, ...
2. Build the Package into a static library `lib<product-name>.a`
3. Find and extract the objectfile from `lib<product-name>.a` that defines the application entrypoint `app_main`. Rename it to something predicatble like `_swiftcode.o`
4. Link `_swiftcode.o` and `lib<product-name>.a` to the IDF build
5. (optional) Export all included header-files to a `compile-flags-txt` for sourcekit-lsp to correctly resolve them

If you are interested in more reading and the process of making this entire thing work, see [Story.md](/Story.md)
