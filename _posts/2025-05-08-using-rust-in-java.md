# Using Rust in Java

Java is fast, and for most use cases the performance is more than enough especially  
considering how mature JVM is, the ecosystem, and how it integrates is enterprise systems. 
But every now and then especially dealing with massive large dataset, the fast enough of 
JVM is not enough espeically in real-time data analytics systems. 

Imagine building a high throughput real-time application that needs  to parse huge json payload every second. 
Jackon -> is the go to library in Java, this starts to feel sluggist. Even Apache Sparj uses Jackon for parsing json. [REFERNCES]. 
We can optimiuze the java code, tweak the JVM flags, increase the heap memroy etc  but still we 
may not be able to break the performance ceiling.

Lets run a benchmarking application to parse json using Jackson. 

<< CODE >> and << RESULT >> 

But on the other side of spectrum there  is Rust, a system langage that is fast, and fine grained control. 
What if we can bring the speed of rust into our JVM application  without rewriting everyting ? 


Lets run a benchmarking applicaiton to parse JSON using Jackson and compare with Rust simd-json. 

```
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

```
use simd_json::OwnedValue;
use std::fs;
use std::time::Instant;

fn main(){
    let data = fs::read("/Users/prajin/Downloads/blogs/codee/twitter_large.json").expect("Failed to load json");
    let mut json_data = data.clone();
    let start = Instant::now();
    let _parsed: OwnedValue = simd_json::to_owned_value(&mut json_data).expect("Failed to parse");
    let duration = start.elapsed();

    println!("Simd JSON parse time {:?}",duration );


}
```

Results: 
Results:

simd-json parse time: 898.583 µs
Jackson parse time: 123 ms

Result: simd-json is ~137x faster than Jackson in this benchmark. 
This blog is not about benchmarking so there might be other ways to incresae the performance on Java,and also I am using older version of java ( java 8), not JVM tweaks, code optimization.  

This performacne boost is not magic. Rust simd-json leverages SIMD(Single Instruction - Multiple Data)
and zero-copy parsing at the lower level than what Java libraires like Jackson does becuase of JVM limitations 
like garbage collector, memory instrucitons, and lack of direct SIMD usage. 

### Can we bring this performacne of Rust our Java Applciation without rewriting weverytinh?
I faced a similar challenge in a JVM-based project. By using JNI (Java Native Interface). 
I integrated a Rust-based parser directly into the Java system with minimal overhead. 
In this blog, I’ll walk you through how to tap into Rust's power using JNI — with a focus on comparing JSON parsing performance using Jackson vs. simd-json.

Whether you're performance-obsessed, building real-time pipelines, 
or just curious to learn something new, 
this post might offer a valuable perspective. 

### World of JVM 

Before jumping into JNI, and rust. Lets quickly talk bout JVM ( Java Virtual Machine) just to understnad why soimetiem JVM hits a wall. 
Java runs on JVM which is brilliant engineering. It gives portabilioty, mermory sfetly, lots of tooling, write once - run anywhere. 
Thaty is why Java is such heavily used in enterprise applciations, and it serves the purpose. 
But JVM is not magic. It has some trade offs. One of them is memory management which is handled through garbage collector. 
This is conveneint for programmers, we do not have to worry about memory leesk, or manually deallocatio mermoy but it also ments  we do not 
have control the full control. 
SIMD - SIngle isntrcution Multiple data allows CPU's to do hiughly parallel processing o nchunks of data. 
Like vector match used in Deep learnign. Rust and C++ heavily use SIMD and take full advantage of this. But JVM not so much 
It abstract lowwer level control. But new versions of Java does summport SIMD through vector API [ References]. 

Ther are tons of great resouces on  these topics to explore. 

## JNI (Java Native Interface)
If you ahve never workd wioth JNI, dont worry you might not work ever as you  might  not need it ever. But it is good to 
have basic idea that it exsits and what it is. 

JNI is a way for  java code or call (or to be called) by native applicatiosn or libraries wirrtgie in langaues liek C, C++, RUst. 
With this we can escale the JVM sandbox environment and run code that can use system level features to squeueze our raw perfdoramcne IF NEEDED. 

At a very high level this is how JNI works:
1. We decralse na native method in Java class. 
2. We rite the implementation of tht native method in an ative langauielike (C, C++,  JRist)
3. Using System.loadLibrary()  we lload the complied native code. 
4. Let JVM pas data between Jva and native coe drugn runtime.  

While this is pwoerful, power comes wth responsibilty. Sicne  it is quite low lvel we havew to amngbe mameroy, carefully,  abd be very 
explicit on wht kind ofdata to pass acorss boundary. 

I am No export inthis topic but feel free to explore more on thistopic if this  is intering to you. 

In our case , we will use JNI to call JSON parsing function written n rust directly from Java. 
We wont go through erach  and every deetail but as we go alogn through the integration. I will  try ot explain how to intergtate.  

We wont dive deep into how simd-json works in rust. In short, it is a json parser writtn in rust that uses 
lowlevel SIMD to process bytes in parallel which makes it extremely fast. If youare curou please head over to the 
official inmplemantion and ssimds json paper. 

## Portign rust json parsr to java 

Lets see an example how are we integrating rust with Java using JNI. The basic strcutuir would be: 
1. Java calls the native method "parseJson(String path)"
2. Rust read the file , parses the json, and returns. 
3. JNI serves as the bridge between them


```
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
The modules Cstr, CString, and c_char are to handle the string conversion between Rust adn C. The 
function parse_json  is marked with #[no_mangle] and extern "C" -> This tells that Rust to expose this function 
without name mangling and to use the C calling  convention so that JNI can call it safelty. 

Naming Convention Note: The name of the Rust function must exactly match what the JVM expects. 
If the Java class is named RustJsonParser and it declares a method native String parseJson(String json);, 
then the Rust function must be named:
Java_RustJsonParser_parseJson

This naming rule is how JNI binds native functions to Java methods at runtime.


The Java Class(RustJsonParser.java) is responsible for bridging the JVM to the native Rust code. 
It loads the compiled shared library (.dylib, .so, or .dll) using System.load(...) and 
declares a native method parseJson(...), which Java expects to find in the loaded library.
This class does not contain the logic for benchmarking or file iteration — its sole responsibility is to provide a thin 
JNI wrapper around the Rust library.

```
public class RustJsonParser {
    static {
        System.load("/absolute/path/to/librust_json_parser.dylib");
    }

    public static native String parseJson(String path);
}

```

#### Java Benchmark App (Main.java)

This class contains the actual logic i.e to laod the jsion file, pass the file to RustJsonParser.

```
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


First we ompile Rust for x86_64-apple-darwin (important for JVM compatibility): [ I am using Mac M1].

```
cd rust_json_parser
rustup target add x86_64-apple-darwin
cargo build --release --target x86_64-apple-darwin

```

The native compiled library "librust_json_parser.dylib" , we need to load this in the Java.  

Results:



