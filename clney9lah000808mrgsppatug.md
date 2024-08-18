---
title: "Edge IoT with Rust on ESP: HTTP Client"
datePublished: Fri Oct 06 2023 18:38:40 GMT+0000 (Coordinated Universal Time)
cuid: clney9lah000808mrgsppatug
slug: edge-iot-with-rust-on-esp-http-client
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1696587512782/eedb7f7b-d258-4b46-a57c-7e1e2e66e22e.png
tags: tutorial, iot, rust, embedded, esp32

---

> ***This blog post is the second of a multi-part series of posts where I explore various peripherals in the ESP32C3 using standard library embedded Rust and the esp-idf-hal. Please be aware that certain concepts in newer posts could depend on concepts in prior posts.***

Prior posts include (in order of publishing):

1. [Edge IoT with Rust on ESP: Connecting WiFi](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-connecting-wifi)
    

## Introduction

In the last post, we understood how to establish a WiFi connection on an ESP device using the Rust ESP-IDF framework. Now that we have access to the network, we are open to a world of possibilities. Access to networks from the application relies heavily on protocols. There are various protocols for various purposes. This series will cover some of the popular ones that are also supported by the ESP framework. If you need a refresher on the Rust framework you can check this post.

HTTP (Hyper Text Transfer Protocol) is a core internet protocol and one of the most utilized. In fact, HTTP is probably the protocol you use most frequently every day as you surf the internet. This is because requests for website HTML objects (or pages) all happen over HTTP. We'll talk a bit about HTTP in more detail later in this post.

In this post, we are going to use HTTP to get a URL using the ESP and Rust.

%%[substart] 

### **📚 Knowledge Pre-requisites**

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Basic familiarity with WiFi & HTTP.
    

### **💾 Software Setup**

All the code presented in this post is available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [**here**](https://wokwi.com/projects/377728497260065793).

### **🛠 Hardware Setup**

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp&auto=compress,format&format=webp&auto=compress,format&format=webp&auto=compress,format&format=webp align="center")
    

## **👨‍🎨 Software Design**

HTTP, or Hypertext Transfer Protocol, serves as the backbone of Internet communication. It follows a client-server model and enables the exchange of data between web browsers and servers. When you enter a website's URL into your browser, it sends an HTTP request to the server hosting that site. This request specifies the action to be performed, like fetching a webpage or submitting data. HTTP also uses the TCP protocol to transfer its data.

HTTP requests use methods like GET, POST, PUT, or DELETE to convey their purpose. The Uniform Resource Locator (URL) identifies the resource to be accessed. Requests and responses can include headers, providing additional information about the data being transmitted.

The server responds with a status code indicating the outcome of the request. For instance, 200 signifies success, 404 denotes a resource not found, and 500 signals a server error. Notably, HTTP is stateless, meaning it treats each request independently. Keep in mind that HTTP lacks encryption, making data vulnerable to interception. To address this, HTTPS encrypts the data transmitted, ensuring secure communication between clients and servers.

A GET request is one of the most common types of HTTP requests used on the internet. It's initiated by a client, often a web browser, to request data from a specified resource on a server. For example, when you enter a website's URL into your browser and press Enter, the browser sends a GET request to the server hosting the website. Here's how a typical request interaction works:

1. **Initiating Connection**: A TCP (Transmission Control Protocol) connection with the server is established.
    
2. **Sending the GET Request**: Once the connection is established, the request is sent to the server. This request includes several headers including the specific resource's URL that you want to access (like a webpage or an image). The client might also send information in the headers such as the browser type (User-Agent) in addition to others.
    
3. **Server Processes the Request**: The web server receives the GET request and interprets it. It locates the requested resource in its storage and prepares to send it back to the client.
    
4. **Server Sends the Response**: After locating the requested resource, the server packages it into an HTTP response. This response includes a status code indicating the outcome of the request (commonly 200 OK for a successful request) and the requested data (like an HTML file for a webpage).
    
5. **Receiving and Rendering the Response**: The client receives the HTTP response containing several headers. If the status code is 200, indicating success, the client processes the received message (For example if an HTML file it displays the webpage's content). If it's a different status code, the client handles the response accordingly (for example, displaying an error page for a 404 status code).
    
6. **Closing the Connection**: After the data is transmitted, the TCP connection is closed. This completes the GET request interaction.
    

In this post, we are going to configure HTTPS and create a request message. Since we're using a headless embedded system (no screen), we're not going to be able to display a page. Instead, we'll print out to the console some of the headers in the response message.

1. Configure and Connect to WiFi
    
2. Configure HTTP and Create an HTTP Client
    
3. Prepare and Submit/Send Request
    
4. Process the HTTP Response
    

## **👨‍💻 Code Implementation**

### **📥 Crate Imports**

In this implementation, the following crates are required:

* The `anyhow` crate for error handling.
    
* The `esp_idf_hal` crate to import the peripherals.
    
* The `esp_idf_svc` and `embedded_svc` crates to import the device services (wifi and HTTP in particular).
    
* The `embedded_svc` crate to import the needed service traits.
    

```rust
use anyhow;
use embedded_svc::http::client::Client;
use embedded_svc::wifi::{AuthMethod, ClientConfiguration, Configuration};
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::http::client::{Configuration as HttpConfig, EspHttpConnection};
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::wifi::{BlockingWifi, EspWifi};
```

### **🎛 Initialization/Configuration Code**

1️⃣ **Obtain a handle for the device peripherals**: Similar to all past blog posts, in embedded Rust, as part of the singleton design pattern, we first have to take the device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `peripherals` as follows:

```rust
let peripherals = Peripherals::take().unwrap();
```

2️⃣ **Configure and Connect to WiFi**: this involves the same steps that were done in the last [post](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-connecting-wifi). However, there are two changes:

1. Wrapping the `EspWiFi` driver with a `BlockingWifi` abstraction:
    
    ```rust
    let mut wifi = BlockingWifi::wrap(
        EspWifi::new(peripherals.modem, sysloop.clone(), Some(nvs))?,
        sysloop,
    )?;
    ```
    
2. Using the `wait_netif_up` method a statement is added that makes sure the network is up before proceeding:
    
    ```rust
    wifi.wait_netif_up()?;
    ```
    

Why these changes? Well, it turns out that as I was working with WiFi and HTTP, there were some network interfaces that had to be up. If they are not, HTTP fails to work. `wait_netif_up` ensures that these interfaces are up before proceeding. However, `wait_netif_up` only is available with the `BlockingWifi` and `AsyncWifi` abstractions. `AsyncWifi` would be useful if blocking behavior is not desired.

3️⃣ **Create the HTTP Connection Handle:** Within `esp_idf_svc::http::client` there exists an `EspHttpConnection` abstraction. This is the abstraction needed to set up and configure an HTTP connection. `EspHttpConnection` contains a `new` method for instantiation that takes a single reference to a `http::client::`[`Configuration`](https://esp-rs.github.io/esp-idf-svc/esp_idf_svc/http/client/struct.Configuration.html#) type parameter. Following that we create an `httpconnection` handle as follows:

```rust
// Create HTTPS Connection Handle
let httpconnection = EspHttpConnection::new(&HttpConfig {
    use_global_ca_store: true,
    crt_bundle_attach: Some(esp_idf_sys::esp_crt_bundle_attach),
    ..Default::default()
})?;
```

Note two things:

1. I am using `HttpConfig` instead of `Configuration` . This is because in the imports I declared `Configuration as HttpConfig` . I did this because the http `Configuration` name overlaps with the wifi `Configuration` name.
    
2. We didn't go for `Default` settings for the `HttpConfig`, but rather defined two members; `use_global_ca_store` and `crt_bundle_attach` . These are required for secure HTTP or HTTPS. Going for `Default` would setup vanilla HTTP. The issue is that there are rarely any websites nowadays that support non-secure HTTP.
    

4️⃣ **Wrap the HTTP Connection in a Client Abstraction:** Since we're going to be operating as a client we need access to the client-related methods. Within the `embedded-svc` traits there exists a `http::client::Client` abstraction. `Client` has a `wrap` method that takes a single `Connection` parameter. The connection we need to pass in this case is the `httpconnection` handle:

```rust
let mut httpclient = Client::wrap(httpconnection);
```

### **📱 Application Code**

1️⃣ **Submit HTTP Request**: Now that both wifi and http are configured, and wifi connected, we need to submit the HTTP request. We're going to send a GET request to [https://httpbin.org](https://httpbin.org). There are two main steps here; preparing the request and submitting the request. To prepare the request there exists the `get` method for `Client` that takes a URL string slice (`&str`) as an argument. We can create a handle named `request` that is associated with the prepared request as follows:

```rust
// Define URL
let url = "https://httpbin.org/get";
// Prepare request
let request = httpclient.get(url)?;
```

`get` returns a `Request` type wrapped in a `Result`. The `?` operator takes care of unwrapping the `Result` if there are no errors. For submitting the request there exists a `submit` method for `Request` that returns a `response` and is used as follows:

```rust
// Submit Request and Store Response
let response = request.submit()?;
```

`submit` returns a `Response` type wrapped in a `Result`. Again, the `?` takes care of the unwrapping.

**2️⃣ Process HTTP Response:** In response to the GET request we should expect some code (200 code hopefully!) along with some headers and a body. We're only going to print some headers to check their values. You can check the MDN web docs [here](https://developer.mozilla.org/en-US/docs/Glossary/Response_header) for some common headers. However, if you want to print out the response, this involves more code to parse the result. `Response` has several methods to retrieve information from `response`. We're going to use two of them; `status` and `header` . `status` takes no arguments but returns a `u16` with the status code of the request. `header` takes one argument which is a `&str` wrapped in an `Option` for the name of the header we want to retrieve. The `Option` would return a `None` if the header requested is not part of the response.

The below code retrieves the status code and some header and prints them to the console:

```rust
let status = response.status();
println!("<- {}", status);

match response.header("Content-Length") {
    Some(data) => {
        println!("Content-Length: {}", name);
    }
    None => {
        println!("No Content-Length Header");
    }
}
match response.header("Date") {
    Some(data) => {
        println!("Date: {}", name);
    }
    None => {
        println!("No Date Header");
    }
}
```

This is it! We've created a simple HTTP client!

## **📱Full Application Code**

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Also, the Wokwi project can be accessed [**here**](https://wokwi.com/projects/377728497260065793).

```rust
use anyhow;
use embedded_svc::http::client::Client;
use embedded_svc::wifi::{AuthMethod, ClientConfiguration, Configuration};
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::http::client::{Configuration as HttpConfig, EspHttpConnection};
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::wifi::{BlockingWifi, EspWifi};

fn main() -> anyhow::Result<()> {
    esp_idf_sys::link_patches();

    // Configure Wifi
    let peripherals = Peripherals::take().unwrap();
    let sysloop = EspSystemEventLoop::take()?;
    let nvs = EspDefaultNvsPartition::take()?;

    let mut wifi = BlockingWifi::wrap(
        EspWifi::new(peripherals.modem, sysloop.clone(), Some(nvs))?,
        sysloop,
    )?;

    wifi.set_configuration(&Configuration::Client(ClientConfiguration {
        ssid: "SSID".into(),
        bssid: None,
        auth_method: AuthMethod::None,
        password: "PASSWORD".into(),
        channel: None,
    }))?;

    // Start Wifi
    wifi.start()?;

    // Connect Wifi
    wifi.connect()?;

    // Wait until the network interface is up
    wifi.wait_netif_up()?;

    // Print Out Wifi Connection Configuration
    while !wifi.is_connected().unwrap() {
        // Get and print connection configuration
        let config = wifi.get_configuration().unwrap();
        println!("Waiting for station {:?}", config);
    }

    println!("Wifi Connected, Intiatlizing HTTP");

    // HTTP Configuration
    // Create HTTPS Connection Handle
    let httpconnection = EspHttpConnection::new(&HttpConfig {
        use_global_ca_store: true,
        crt_bundle_attach: Some(esp_idf_sys::esp_crt_bundle_attach),
        ..Default::default()
    })?;

    // Create HTTPS Client
    let mut httpclient = Client::wrap(httpconnection);

    // HTTP Request Submission
    // Define URL
    let url = "https://httpbin.org/get";

    // Prepare request
    let request = httpclient.get(url)?;

    // Log URL and type of request
    println!("-> GET {}", url);

    // Submit Request and Store Response
    let response = request.submit()?;

    // HTTP Response Processing
    let status = response.status();
    println!("<- {}", status);

    match response.header("Content-Length") {
        Some(data) => {
            println!("Content-Length: {}", data);
        }
        None => {
            println!("No Content-Length Header");
        }
    }
    match response.header("Date") {
        Some(data) => {
            println!("Date: {}", data);
        }
        None => {
            println!("No Date Header");
        }
    }

    Ok(())
}
```

## Conclusion

HTTP is a core protocol in Internet applications. As a result, it forms a crucial aspect of IoT projects and enables a wide variety of applications. ESPs also some of the most popular devices among makers for enabling such projects. This post introduced how to create an HTTP client on ESP using Rust and the `esp_idf_svc` crate. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]