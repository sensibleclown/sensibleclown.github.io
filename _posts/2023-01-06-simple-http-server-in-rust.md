---
title: Simple HTTP Server in Rust
date: 2023-01-06 08:00:00 +/-TTTT
categories: [programming, sample-code]
tags: [rust, http-server] # TAG names should always be lowercase
---

Here's an example of an HTTP server in Rust that uses the std::net and std::io libraries:

```rust
use std::net::{TcpListener, TcpStream};
use std::io::{Read, Write};
use std::thread;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:3000").unwrap();

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        thread::spawn(|| {
            handle_connection(stream);
        });
    }
}

fn handle_connection(mut stream: TcpStream) {
    let mut buffer = [0; 512];
    stream.read(&mut buffer).unwrap();

    let response = "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\nContent-Length: 12\r\n\r\nHello, World!";

    stream.write(response.as_bytes()).unwrap();
    stream.flush().unwrap();
}
```

This server listens on http://127.0.0.1:3000, and responds to all requests with a 200 OK response with a body of "Hello, World!".

To run this server, you don't need to add any additional dependencies to your Cargo.toml file. You should be able to build and run the server with cargo run.
