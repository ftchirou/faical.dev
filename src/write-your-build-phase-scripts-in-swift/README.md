# Write your Xcode build phase scripts in Swift

In non-trivial iOS/macOS projects, it's not uncommon to have one or more build phases that execute some scripts. The de-facto language for writing those scripts is Bash or Bash-like (sometimes Python or Perl). However, if you're already a Swift developer and you have a *non-trivial script* to run as a build phase, Swift is a perfectly fine and arguably better alternative. 

When writing Swift scripts, beside the niceness of the language and the type-safety it affords, we have access to powerful Foundation APIs for performing network requests, encoding and decoding JSON, writing concurrent code, accessing the file system, handling errors, etc. 

While it's easy enough to write build phase scripts in Swift, we'll explore some good practices and a few considerations and gotchas to keep in mind.

## The basics

A basic Swift script looks like the following.

```swift
#!/usr/bin/env xcrun swift

// MyScript.swift

print("Hello, World!")
```

We can execute this script using the Swift interpreter.

```shell
$ swift MyScript.swift
```

Or we can make the file executable, then execute it.

```shell
$ chmod u+x MyScript.swift
$ ./MyScript.swift
```

My favorite though, is to first compile the script with `swiftc`, then run the generated executable.

```shell
$ swiftc MyScript.swift
$ ./MyScript
```

I prefer this approach because some versions of `swift` [crash with a segmentation fault](https://bugs.swift.org/browse/SR-12403) while executing scripts that import `Foundation`. Compiling the script instead of directly executing it ensures that our scripts are "portable" (when used as build phase scripts) across different versions of Xcode.

One gotcha to watch out for in iOS projects is that the default SDK used to compile (and/or run) a Swift script in an Xcode build phase is the iOS SDK. Your script will fail to compile/run if you use macOS-only APIs. Since our scripts are meant to run on macOS anyway, a more appropriate way to invoke them is to use `xcrun` and specify the macOS SDK.

```shell
$ xcrun --sdk macosx swiftc MyScript.swift
$ ./MyScript
```

Our basic Xcode build phase Swift script should now look like the following.

<img src="../assets/images/basic-script.png" alt="A basic build phase script in Swift.">

## Providing an entry point

While it's perfectly possible to write all of your script's code at the top-level of the file (à la Bash), for more complex scripts, it would be preferable (for readability and navigability) to break the script down into entities and functions, and to provide a clear entry point.

As of Swift 5.3, we can annotate an entity with the `@main` attribute to specify that the static function named `main` should be the entry point of the program. We can apply that to our script.

```swift
@main
struct MyScript {
  static func main() {
    print("Hello, World!")
  }
}
```

This leads us to our second gotcha. Trying to build the project containing this script yields the following error.

```shell
MyScript.swift:1:1: error: 'main' attribute cannot be used in a module that contains top-level code
@main
^
MyScript.swift:1:1: note: top-level code defined in this source file
@main
^
```

This seems to be a [bug](https://bugs.swift.org/browse/SR-12683) related to how the Swift compiler detects top-level code when it is passed a single file to compile. One workaround is to tell the compiler to parse the file as a library. Our script invocation then becomes:

```shell
$ xcrun --sdk macosx swiftc -parse-as-library MyScript.swift
$ ./MyScript
```

<img src="../assets/images/script-with-main.png" alt="A build phase script in Swift containing a main entry point.">

Using `-parse-as-library` by default will ensure that our script is portable across versions of Xcode even if this bug is fixed in a future version of the compiler.

Of course, we also have the option to not use the `@main` attribute and call the `main` function explicitly (especially if compatibility with Swift versions lower than `5.3` is required).

```swift
struct MyScript {
  static func main() {
    print("Hello, World!")
  }
}

MyScript.main()
```


## Logging messages

An important part of any computer program is to communicate to the outside world what actions are being taken, what events are being triggered, and what errors are occurring. Scripts are no exception. While we can simply use `print` to display messages, we can go one step further by using [`FileHandle`](https://developer.apple.com/documentation/foundation/filehandle) to write information and success messages to the standard output file and error messages to the standard error file.

```swift
import Foundation

@main
struct MyScript {
  static func main() {
    Log.info("Hello, World!")
  }
}

enum Log {
  static func info(_ message: String) {
    FileHandle.standardOutput.write("[INFO] \(message)\n".utf8Data)
  }

  static func success(_ message: String) {
    FileHandle.standardOutput.write("[SUCCESS] \(message)\n".utf8Data)
  }

  static func failure(_ message: String) {
    FileHandle.standardError.write("[FAILURE] \(message)\n".utf8Data)
  }
}

// ...
```

Now we can invoke our script using the following.

```shell
$ xcrun --sdk macosx swiftc -parse-as-library MyScript.swift
$ ./MyScript > MyScriptMessages.log 2> MyScriptErrors.log
```

The information and success messages will be stored in `MyScriptMessages.log` and the error messages in `MyScriptErrors.log`.

For a fine-grained control of the location of these logs, we can use the "Output Files" section of the build phase in Xcode. As a result, our build phase will look like the following.

<img src="../assets/images/script-with-output.png" alt="A build phase script in Swift with logging capabilities">

(We also used the "Input Files" section to provide the location of the Swift script.)

## Exiting properly

At the end of the execution of our script, we need to communicate to the outside world whether *the execution was successful or ended in a failure*. This is important in situations where subsequent build phases should not be executed by Xcode in case of a failure of our script.

To exit our script and communicate to Xcode that the execution ended successfully, we can use the function `exit` and pass `0` as argument. To communicate that the execution ended in a failure, we pass an integer other than `0` to `exit`. It's worth adding helper functions for clarity.


```swift
import Foundation

@main
struct MyScript {
  static func main() {
    Log.info("Hello, World!")
    exitWithSuccess()
  }
}

// ...

func exitWithSuccess() {
  exit(0)
}

func exitWithFailure() {
  exit(1)
}
```


Whenever `exitWithFailure()` is called during the execution of our script, Xcode will stop the overall build process and report a failure.

## Putting it all together

Let's use all of the above to write a simple build phase script in Swift. For example, a script that concurrently downloads a bunch of JSON resources and persists them in the built products directory.

```swift
import Foundation

@main
struct MyScript {
  static func main() {
    guard let builtProductsDir = ProcessInfo.processInfo.environment["BUILT_PRODUCTS_DIR"] else {
      Log.failure("could not detect the built products directory.")
      exitWithFailure()
      return
    }

    let downloader = Downloader()
    let urls = ["https://{resource}", "https://{resource}", "https://{resource}"].compactMap { URL(string: $0) }
    let dispatchGroup = DispatchGroup()

    for url in urls {
      dispatchGroup.enter()

      downloader.download(from: url) { data in
        defer { dispatchGroup.leave() }

        guard let data = data else {
          return
        }

        let output = URL(fileURLWithPath: "\(builtProductsDir)/Resources/{resource}.json")

        do {
          try data.write(to: output)
          Log.success("wrote resource to file \(output.absoluteString).")
        } catch {
          Log.failure("failed to write resource to file \(output.absoluteString): \(error.localizedDescription).")
        }
      }
    }

    dispatchGroup.notify(queue: .main) {
      Log.info("done!")
      exitWithSuccess()
    }
  }
}

struct Downloader {
  func download(from url: URL, completion: @escaping (Data?) -> Void) {
    let request = URLRequest(url: url, cachePolicy: .useProtocolCachePolicy)

    URLSession.shared.dataTask(with: request) { data, response, error in
      if let error = error {
        Log.failure("failed to download from \(url.absoluteString): \(error.localizedDescription).")
        return completion(nil)
      }

      guard let response = response as? HTTPURLResponse, let data = data else {
        Log.failure("failed to download from \(url.absoluteString): no data received.")
        return completion(nil)
      }

      guard response.statusCode.isSuccess else {
        Log.failure(
          """
          failed to download from \(url.absoluteString).
            - status code: \(response.statusCode)
            - response: \(String(data: data, encoding: .utf8) ?? "")
          """
        )
        return completion(nil)
      }

      Log.success("successfully downloaded resource from \(url.absoluteString).")
      completion(data)
    }.resume()
  }
}

// ...
```

Depending on the importance of our script, we could update the invocation of our script to automatically open the log files after the execution.

<img src="../assets/images/final-script.png" alt="A simple and complete build phase Swift script.">

Compared to a Bash equivalent, the script above is more verbose but it is more performant, maintainable and easy to read and understand for Swift developers.

Thanks for reading 👋.