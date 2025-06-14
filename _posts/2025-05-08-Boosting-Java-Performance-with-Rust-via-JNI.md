Java is fast, and for most use cases, the performance is more than enough — 
especially considering how mature the JVM is, the ecosystem around it, and how well it integrates into enterprise systems.
But every now and then, especially when dealing with **massive datasets**, 
the "fast enough" of the JVM is not enough — particularly in real applications.

Imagine building a high-throughput, real-time application that needs to parse **huge JSON payloads every second**.
[Jackson](https://github.com/FasterXML/jackson) is the go-to library in Java, 
and even [Apache Spark uses Jackson for parsing JSON](https://github.com/apache/spark/blob/master/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/json/JacksonParser.scala).
However, Jackson starts to feel sluggish when dealing with large JSON files.

Obviously, we can optimize the Java code, tweak the JVM flags, 
increase heap memory, etc., but we may still not be able to break the performance ceiling.

On the other there's **Rust** — 
a systems language that is fast and offers fine-grained control.

What if we could **bring Rust's speed** into our JVM application without rewriting everything?

Let's run a simple benchmarking application to parse JSON using Jackson in Java and 
compare it with `simd-json` in Rust to see if Rust performs better or worse.

## Java Benchmark

```java
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.nio.file.Files;
import java.nio.file.Paths;

public class JacksonBenchmark {
    public static void main(String[] args) throws Exception {
        byte[] jsonData = Files.readAllBytes(Paths.get("twitter_large.json"));
        ObjectMapper mapper = new ObjectMapper();
        long startTime = System.nanoTime();
        JsonNode rootNode = mapper.readTree(jsonData);
        long endTime = System.nanoTime();
        System.out.println("Jackson parse time: " + (endTime - startTime) / 1_000_000 + " ms");
        System.out.println("Parsed JSON root node: " + rootNode.getNodeType());
    }
}
```

## Rust Benchmark

```rust
use simd_json::OwnedValue;
use std::fs;
use std::time::Instant;

fn main() {
    let data = fs::read("twitter_large.json").expect("Failed to load JSON");
    let mut json_data = data.clone();
    let start = Instant::now();
    let _parsed: OwnedValue = simd_json::to_owned_value(&mut json_data).expect("Failed to parse");
    let duration = start.elapsed();

    println!("Simd JSON parse time {:?}", duration);
}
```

```
simd-json parse time: 898.583 µs
Jackson parse time: 123 ms
```

`simd-json` is ~137x faster than `Jackson` in this benchmark.

*This blog is not about benchmarking, so there might be other ways to improve Java's performance. 
I'm also using an older version of Java (Java 8), with no JVM tweaks or code optimization.*

This performance boost isn't magic.
`simd-json` in Rust leverages SIMD (Single Instruction, Multiple Data) and zero-copy parsing at 
a lower level than what Java libraries like `Jackson` can achieve — primarily because of JVM limitations like garbage collection, 
memory instruction abstraction, and lack of direct SIMD usage.

**Can we bring Rust's performance to our Java application without rewriting everything?**

I faced a similar challenge in a JVM-based project.
By using JNI (Java Native Interface), I integrated a Rust-based parser directly into the Java system with minimal overhead.

In this blog, I'll walk you through how to tap into Rust's power using JNI — with a focus on comparing JSON 
parsing performance using `Jackson` vs. `simd-json`. Json parsing may not be the best use case here.
This specific example(Json Parser) is just used here to show the usage of JNI.


## World of JVM

Before jumping into JNI and Rust, 
let's quickly discss about the JVM (Java Virtual Machine) just to understand why it sometimes hits a wall.

Java runs on the JVM, which is a brilliant piece of engineering.
It provides portability, memory safety, great tooling, and the "write once, run anywhere" capability.

That's why Java is so heavily used in enterprise applications — and it serves well.

But the JVM is not magic. It has trade-offs. One of them is memory management, handled via the Garbage Collector (GC).
This is convenient for programmers — we don't have to worry about memory leaks or manual deallocation — 
but it also means we do not have full control. 

SIMD (Single Instruction, Multiple Data) allows CPUs to do highly parallel 
processing on chunks of data — like vector math used in deep learning. Rust and C++ leverage SIMD effectively. JVM? Not so much.
It abstracts away lower-level control. That said, newer versions of Java support SIMD through the Vector API.

## JNI (Java Native Interface)

If you've never worked with JNI — don't worry, you might never need it.
But it's good to know it exists and understand what it does. JNI is a way for Java code to call (or be called by) 
native applications or libraries written in languages like C, C++, or Rust. With JNI, we can escape the JVM sandbox and 
run code that can use system-level features to squeeze out raw performance — if needed.

At a high level, JNI works like this:

1. Declare a native method in the Java class.
2. Write the implementation of that method in a native language (C, C++, Rust).
3. Use System.loadLibrary() to load the compiled native code.
4. Let the JVM pass data between Java and native code during runtime.

While powerful, this comes with responsibility.
Since it's quite low-level, we have to manage memory carefully and be explicit about what kind of data we pass across the boundary.

## Porting a Rust JSON Parser to Java via JNI

The basic structure is:

1. Java calls the native method `parseJson(String path)`
2. Rust reads the json file, parses the JSON, and returns.
3. JNI serves as the bridge between them

##  Rust Code

```rust
use std::fs;
use jni::objects::{JClass, JString};
use jni::sys::jstring;
use jni::JNIEnv;
use simd_json::to_owned_value;

#[no_mangle]
pub extern "system" fn Java_RustJsonParser_parseJson(
    mut env: JNIEnv,
    _class: JClass,
    jpath: JString,
) -> jstring {
    let path: String = match env.get_string(&jpath) {
        Ok(p) => p.into(),
        Err(_) => return std::ptr::null_mut(),
    };

    let data = match fs::read(&path) {
        Ok(d) => d,
        Err(_) => return null_string(&env, "error"),
    };

    let mut json_data = data.clone();
    let result = match to_owned_value(&mut json_data) {
        Ok(_) => "parsed",
        Err(_) => "error",
    };

    match env.new_string(result) {
        Ok(jstr) => jstr.into_raw(),
        Err(_) => std::ptr::null_mut(),
    }
}

fn null_string(env: &JNIEnv, msg: &str) -> jstring {
    env.new_string(msg).unwrap_or_default().into_raw()
}
```
`CStr`, `CString`, and `c_char` helps with string conversion between Rust and C.

The function `Java_RustJsonParser_parseJson` is marked with:

- `#[no_mangle]` to avoid name mangling
- `extern "system"` to use the C calling convention so JNI can call it safely

If the Java class is named `RustJsonParser` and it declares a method `native String parseJson(String json);`,
then the Rust function must be named:

`Java_RustJsonParser_parseJson`

This naming rule is how JNI binds native functions to Java methods at runtime.

## RustJsonParser.java

This class bridges the JVM to native Rust code.
It loads the compiled shared library (.dylib, .so, or .dll) using `System.load(...)` and declares the 
native method.

```java
public class RustJsonParser {
    static {
        System.load("/absolute/path/to/librust_json_parser.dylib");
    }
    public static native String parseJson(String path);
}
```

This class does not contain any benchmarking logic — it only provides a thin JNI 
wrapper around the Rust library.

## Main.java

This class contains the actual logic to load the JSON file and pass the file path to the Rust parser.

```java
import java.io.*;

public class Main {
    public static void main(String[] args) {
        String filePath = "../test_inputs/twitter_large.json";
        long start = System.nanoTime();
        String result = RustJsonParser.parseJson(filePath);
        long end = System.nanoTime();
        long totalTime = end - start;
        System.out.println("\nResult: " + result);
        System.out.printf("Total time: %.2f ms\n", totalTime / 1_000_000.0);
    }
}
```

Since I'm on a Mac M1, I compiled Rust for `x86_64-apple-darwin`

```
cargo build --release --target x86_64-apple-darwin 
```

Now, the native compiled library `librust_json_parser.dylib` can then be loaded into the Java application.

The test JSON file used here is the [twitter.json](https://github.com/serde-rs/json-benchmark/blob/master/data/twitter.json) 
from the JSON benchmark repository, duplicated over 1000 times to reach a file size of approximately 561MB. When parsing this file 
with Jackson in Java, it takes around 9000ms, with the JVM heap memory explicitly set to 4GB using `-Xmx4g`.

The Java + Rust JNI integration shown above brings that down to around 6000ms, which is a 
noticeable improvement. That said, 
the Rust code here does not return the parsed JSON to Java — it simply parses 
and returns a placeholder string ("parsed"). 
This omission avoids serialization and JNI overhead, and is not reflective of a full end-to-end use case. 
Returning large data structures from Rust to Java would require additional complexity 
and may reduce the observed performance gain.

Lastly, this post is not meant to be a benchmark authority — instead, 
it's focused on demonstrating how to call Rust code from 
Java using JNI, particularly for performance-critical components.

All the code that I have used are hosted here: [Rust-In-Java](https://github.com/prajinkhadka/using-rust-java)

Notes:
* Latest version of Java do support SIMD through a Vector API from Java 16, and is a 
standard feature through Java 17.
* [SIMDjson-Java](https://github.com/simdjson/simdjson-java) 

