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

I was facing a similar kind of problem while I was workinbg on a project tht bas hevily based on JVM.  
I used JNI to natively use rust from java minziming soem overhead on JVM. In this blog, I will be shairng 
how can we tap into rust power using JNI(Java Native Interface) with an example: Parsing Large Json comparinsg 
Jackson parser and simd-json from rust showing how to bridge betwee two world, and analyzing the performance gain. 

Whether you are performacne obsessed or just curious to learn somethin new this mpost might be valuaable. 

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



