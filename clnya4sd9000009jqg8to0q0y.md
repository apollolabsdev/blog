---
title: "Edge IoT with Rust on ESP: HTTP Server"
datePublished: Fri Oct 20 2023 07:18:29 GMT+0000 (Coordinated Universal Time)
cuid: clnya4sd9000009jqg8to0q0y
slug: edge-iot-with-rust-on-esp-http-server
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1697785911138/a2531290-16dc-47e3-be8b-ad432e07041d.png
tags: tutorial, iot, beginners, rust, esp32

---

> ***This blog post is the third of a multi-part series of posts where I explore various peripherals in the ESP32C3 using standard library embedded Rust and the esp-idf-hal. Please be aware that certain concepts in newer posts could depend on concepts in prior posts.***

Prior posts include (in order of publishing):

1. [Edge IoT with Rust on ESP: Connecting WiFi](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-connecting-wifi)
    
2. [Edge IoT with Rust on ESP: HTTP Client](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-http-client)
    

## Introduction

In the last post, we understood how to create an HTTP client on an ESP device using the Rust ESP-IDF framework. That also required a connection to WiFi that was established and explained in an earlier post. In this post, we are going to use HTTP to create a server using ESP and Rust. For that, we'll create a simple html page that will be returned by a `GET` request from a client.

%%[substart] 

### **📚 Knowledge Pre-requisites**

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Basic familiarity with WiFi & HTTP.
    
* Basic familiarity with HTML.
    

### **💾 Software Setup**

All the code presented in this post is available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [**here**](https://wokwi.com/projects/379086197445130241).

### **🛠 Hardware Setup**

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp&auto=compress,format&format=webp&auto=compress,format&format=webp&auto=compress,format&format=webp align="center")
    

## **👨‍🎨 Software Design**

In the post last week, we mentioned that HTTP follows a client-server model and enables the exchange of data between web browsers and servers. When you enter a website's URL into your browser, it sends an HTTP request to the server hosting that site. This request specifies the action to be performed, like fetching a webpage or submitting data. Also, HTTP requests use methods like GET, POST, PUT, or DELETE to convey their purpose. Additionally, the Uniform Resource Locator (URL) identifies the resource to be accessed.

When a server receives a request, it responds with a status code indicating the outcome of the request. For instance, 200 signifies success, 404 denotes a resource not found, and 500 signals a server error. Also, a `GET` request is one type of HTTP request that's initiated by a client to request data/objects from a specified resource on a server. In the context of websites, on a server, typically the first object requested is the page base HTML object/file. The base HTML file would then include any other in-page elements that the client has to retrieve subsequently. We've covered the client-side interaction last post. Here's how a typical request interaction works on the server side:

1. **Listen for Connections:** A server is an "always on" entity. This is because it has to be available at all times to respond to client requests. Additionally, a server would have a static IP address that it can be reached on. This is not necessarily true for a client. As such, the server keeps on "listening" for any connection requests coming in.
    
2. **Establish Connection:** We've seen that clients have to establish a connection first before exchanging information with a server. Once a request comes in, the server opens an HTTP port establishing a connection with the client.
    
3. **Process and Send Response:** The server locates the requested resource and packages it into an HTTP response. This response includes a status code indicating the outcome of the request (commonly 200 OK for a successful request) and the requested data (like an HTML file for a webpage). If the resource is not located typically the server would respond with a 404 Not Found.
    
4. **Closing the Connection**: After the data is transmitted, the connection is closed. This completes the GET request interaction.
    

In this post, we are going to configure the ESP to operate as an HTTP server responding with a simple HTML page. The steps include the following:

1. Configure and Connect to WiFi
    
2. Configure HTTP and Create an HTTP Server Listener
    
3. Configure a Handler Function that Responds to Incoming Requests
    

## **👨‍💻 Code Implementation**

### **📥 Crate Imports**

In this implementation, the following crates are required:

* The `anyhow` crate for error handling.
    
* The `esp_idf_hal` crate to import the peripherals.
    
* The `esp_idf_svc` and `embedded_svc` crates to import the device services (wifi and HTTP in particular).
    
* The `embedded_svc` crate to import the needed `http` and `wifi` service traits.
    
* The `std::thread` and `std::time` to introduce delays.
    

```rust
use anyhow;
use embedded_svc::http::Method;
use embedded_svc::wifi::{AuthMethod, ClientConfiguration, Configuration};
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::http::server::{Configuration as HttpServerConfig, EspHttpServer};
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::wifi::{BlockingWifi, EspWifi};
use std::{thread::sleep, time::Duration};
```

### **🎛 Initialization/Configuration Code**

1️⃣ **Obtain a handle for the device peripherals**: Similar to all past blog posts, in embedded Rust, as part of the singleton design pattern, we first have to take the device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `peripherals` as follows:

```rust
let peripherals = Peripherals::take().unwrap();
```

2️⃣ **Configure and Connect to WiFi**: this involves the same steps that were done in the last [post](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-http-client).

3️⃣ **Create the HTTP Connection Handle:** Within `esp_idf_svc::http::server` there exists an `EspHttpServer` abstraction. This is the abstraction needed to set up and configure an HTTP server. `EspHttpServer` contains a `new` method for instantiation that takes a single reference to a `http::server::`[`Configuration`](https://esp-rs.github.io/esp-idf-svc/esp_idf_svc/http/server/struct.Configuration.html) type parameter. We're going to go for the `default` configuration. Following that we create a `httpserver` handle as follows:

```rust
// Create Server Connection Handle
let mut httpserver = EspHttpServer::new(&HttpServerConfig::default())?;
```

Note I am using `HttpServerConfig` instead of `Configuration` . This is because in the imports I declared `Configuration as HttpServerConfig` . I did this because the `wifi::Configuration` name overlaps with the wifi `http::server::Configuration` name.

That's it for Configuration!

### **📱 Application Code**

1️⃣ **Define Response Behaviour:** Within the `EspHttpServer` abstraction there is an `fn_handler` method that is used to define response behavior. `fn_handler` takes three parameters; a `uri`, a `Method` , and a closure that includes the request object response. For the `uri` we need to define the path of the object. Here we are only going to serve a webpage for the root path `/` . Also, we're only going to respond to `Get` requests:

```rust
// Define Server Request Handler Behaviour on Get for Root URL
httpserver.fn_handler("/", Method::Get, |request| {
    // Retrieve html String
    let html = index_html();
    // Respond with OK status
    let mut response = request.into_ok_response()?;
    // Return Requested Object (Index Page)
    response.write(html.as_bytes())?;
    Ok(())
})?;
```

There remains the third argument which is the closure. This is where the response behavior is defined for the root HTML object request. There are four things happening here:

1. The HTML object to respond with is retrieved as a `String`. `index_html` is a function that contains and returns an HTML file as a formatted `String`. `index_html` is shown later in this post.
    
2. An OK `response` is constructed from the `request` using the `into_ok_response` method.
    
3. The request is responded to using the `write` method on `response`. `write` expects a slice of `u8` or `&[u8]`, therefore the usage of the `as_bytes` method on the `html` handle `String`.
    
4. An `Ok` is propagated to confirm the request has been completed successfully.
    

> Note here that the `EspHttpServer` abstraction is taking care of several things for us. This includes opening and closing connections and responding with a 404 if a resource isn't found.

2️⃣ **Define the HTML Object:** Recall we retrieved the HTML object as a `String` using the `index_html` function. As such, we have to define this function. You can use the `format!` macro also:

```rust
fn index_html() -> String {
    format!(
        r#"
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <title>esp-rs web server</title>
    </head>
    <body>
    Hello World from ESP!
    </body>
</html>
"#
    )
}
```

`r#` is used with `String` literals to negate the need to escape special characters inside the string. More on it [here](https://doc.rust-lang.org/reference/tokens.html#raw-string-literals).

3️⃣ **Keep the Program Running:** As mentioned earlier, the server needs to stay on. This means that we have to keep the program running so that it doesn't terminate. You might think the use of `loop{}` is sufficient. However, if you try to run the application that way, you'll face a problem. A watchdog will kick in. To avoid that behaviour you can use a simple `sleep` function from `std::thread::sleep`.

```rust
loop {
   sleep(Duration::from_millis(1000));
}
```

## 🏃‍♂️Running the Example

To run the example to access the webpage, you need to use the IP address of the ESP on your network. You can get that from the console output. In the below figure an example is shown. The IP address you need is the `sta ip` and it's `192.168.0.126` .

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1697784140674/2151b4b3-9ce1-48c1-aae0-17737e9c5a83.png align="center")

Additionally, if you are going to run the example on Wokwi, then you need a [private gateway](https://docs.wokwi.com/guides/esp32-wifi).

## **📱Full Application Code**

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Also, the Wokwi project can be accessed [**here**](https://wokwi.com/projects/379086197445130241).

```rust
use anyhow;
use embedded_svc::http::Method;
use embedded_svc::wifi::{AuthMethod, ClientConfiguration, Configuration};
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::http::server::{Configuration as HttpServerConfig, EspHttpServer};
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::wifi::{BlockingWifi, EspWifi};
use std::{thread::sleep, time::Duration};

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

    println!("Wifi Connected, Starting HTTP Server");

    // HTTP Configuration
    // Create HTTP Server Connection Handle
    let mut httpserver = EspHttpServer::new(&HttpServerConfig::default())?;

    // Define Server Request Handler Behaviour on Get for Root URL
    httpserver.fn_handler("/", Method::Get, |request| {
        // Retrieve html String
        let html = index_html();
        // Respond with OK status
        let mut response = request.into_ok_response()?;
        // Return Requested Object (Index Page)
        response.write(html.as_bytes())?;
        Ok(())
    })?;

    // Loop to Avoid Program Termination
    loop {
        sleep(Duration::from_millis(1000));
    }
}

fn index_html() -> String {
    format!(
        r#"
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <title>esp-rs web server</title>
    </head>
    <body>
    Hello World from ESP!
    </body>
</html>
"#
    )
}
```

## Conclusion

HTTP is a core protocol in Internet applications. As a result, it forms a crucial aspect of IoT projects and enables a wide variety of applications. ESPs also some of the most popular devices among makers for enabling such projects. This post introduced how to create an HTTP server on ESP using Rust and the `esp_idf_svc` crate. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]