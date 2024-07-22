# The journey

Seeing the Apple announcement for Embedded Swift sparked an idea. Having a modern ecosystem of development tools, a powerful, intuitive and beautiful high-level programming language in the realm of embedded programming sounded like a far fetched dream. Working with the traditional C/C++ Frameworks for building embedded projects felt dull in comparison, even with the empowering thrill of a working blink-sketch on a microcontroller.

To see Matter, the open standard for smart home appliances, in the same announcement caught me off guard. My dream turned into a firework of possibilities. So I turned to it and tried to make my dream come true (or at least go a step in the direction of its glare).

Having little to no experience with build systems, and more importantly, no idea what would be necessary, I started jumping into the CLI help for `swift build`, the documentation on ESP-IDFs build system and cmake documentation.

## Compiling

Inserting non-C code into ES-IDF is not unheard of. It is trivial by comparison if you can get hold of a `.o`-file. Three cmake-commands is all it takes _if_ you have a way to get said `.o` file.

So, swift needs to generate a single `.o`-file from all `.swift`-files (including dependencies). In `swiftc` this is accomplished with the `-c` flag. Unfortunately, `swift build` is unable to do so (as far as I know) and getting `swiftc` to correctly manage all the "package stuff" (dependencies, access control, ...) did not look possible; `swift build` it is then. What it can do is building all `.swift` files into `.o` files and bundling them archive (`.a`) (from now on called `swiftcode`).

Linking an archive to the IDF build is possible. It gets appended to the archive list and IDF happily builds and links the project. However, new issues arise when working with external components.

> DISCLAIMER: I have a very reduced understanding of the compilation process, especially linking. As such, the following passage might be highly inaccurate and abscent of nuance.

Because archive files are treated by the linker as, well, archives for maybe-required-functions, they are discarded if no symbols are required from them. Somewhere inside the build-process this seems to be the case, as I was unable to "just append" the swift archive when working with the `led_strip` component. This was probably due to the order of the archive files, which by that point was
```
ld [...] -lesp_stdlib -lfreertos   -lespressif__led_strip   -lswiftcode 
         ^            ^            ^                        ^
         ESP builtin libraries     ESP Extra Components     Our Swiftcode
```
In this order, ld was unable to link symbols from `swiftcode` to the earlier `espressif_led_strip` and threw an "unknown reference" error.

Okay, maybe we can solve the issue by prepending our `swiftcode` to the archive list
```
ld [...] -lswiftcode   -lesp_stdlib -lfreertos   -lespressif__led_strip   
         ^             ^                         ^
         Our Swiftcode ESP builtin libraries     ESP Extra Components
```
But now, `freertos` has a problem finding the `app_main` entrypoint. What might work is inserting our `swiftcode` right between the builtin archives and the extra ones.
This would require figuring out where builtins end and extra components start. Doing this with cmake was no option for me (who has basically zero experience working with cmake).

I chose a different approach. Instead of finding the correct archive order, so the linker would not prematurely discard symbols, I linked the objectfiles directly. Symbols in objectfiles do not get removed by the linker if they are not (yet) used. Huraay, a possible solution.

Dumping all objectfiles from our `swiftcode` into the IDF-build revealed another Problem: Duplicate Symbols. Swift apparently includes all stdlib symbols it needs into every objectfile it produces. With archives this is no big problem because the linker just picks the first one it finds when they are needed. When including them as objectfiles directly however, the linker tries to keep them all, resulting in duplicate symbols.

Using the archive as-is does not work but using its entire content doesn't work either. How about only using the objectfile that is required during the entire linking process and leaving the rest as a archive for the linker to lookup later? This would require finding the right objectfile in our `swiftcode`, the one that defines the `app_main`-symbol, and linking this objectfile to our build. After fiddling with the appropriate commands, this was automated into the cmake build process via custom targets/commands. Et voila, a working compilation toolchain that works with Swift Packages.

## Finishing touches

From there on, the functionality was done, time to make it easier maintainable. Insight the `CMakeLists.txt`, kind of at the top there are a few variables that control critical parts of the build process. If you were to change the name of the Product for example, you would change the `SWIFT_PRODUCT_NAME`.

Editing code, especially with many dependencies, is no fun without the help of a LSP, that checks for typos, gives context informations, autocomplete and lookups. sourcekit-lsp however, was not happy with the configuration that I had going at that point. Looking into the console revealed two issues: First, it was unable to find other modules, even if they were defined within the same `Package.swift`. Second, it was unable to build the C bridging modules, because it could not find the include files. The second problem seems easier to fix, so lets start with that. ESP-IDFs include structure is very non-trivial, where components are able to define multiple locations with header files. Nothing to easily tap into. During the cmake phase however, we already pass all the include-directories to `swift build`, so why not extract them into a separate file for sourcekit-lsp to use? Doing so make sourcekit-lsp happy with regards to the C bridging module.

The Swift Modules however, it still refused to find. Reading into some forum posts, it seems, sourcekit-lsp uses fragments of the build process to provide its information. Looking into a fresh Swift Package reveals that it compiles the package with a `x86_64-unknown-linux-gnu` target and a `debug` configuration despite specifying a triple of `riscv32-none-none-eabi` to `sourcekit-lsp -Xswiftc`. And indeed, after creating a symlink from our compilation build folder to this seemingly default location makes sourcekit-lsp happy.

## Admiring the finished work

We are now able to create normal Swift Packages with features such as local or remote dependencies and bridging headers, and cross-compile them within a microcontroller toolchain. Additionally, we are able to interface with the provided APIs and even module system (ESP Components) of foreign frameworks/toolchains.

On top of that we made the development experience on-par with regular (machine-native) Swift development. With this approach and the integration of Swift PM into the landscape of embedded software development, building embedded applications or swift-native libraries as well as managing project structure becomes second-nature to the swift ecosystem. I am truly excited where Swift and especially Swift Embedded is headed in the future.
