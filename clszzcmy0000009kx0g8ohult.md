---
title: "ESP Embedded Rust: Ping CLI App Part 2"
datePublished: Sat Feb 24 2024 11:10:43 GMT+0000 (Coordinated Universal Time)
cuid: clszzcmy0000009kx0g8ohult
slug: esp-embedded-rust-ping-cli-app-part-2
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1708772692067/088c97c1-52ce-4f1d-8584-3d936b18f5c8.png
tags: tutorial, cli, rust, esp32, embedded-systems

---

## Introduction

In this week's post, we're going to add the remainder of the features to the [CLI app last week](https://apollolabsblog.hashnode.dev/esp-embedded-rust-ping-cli-app-part-1). This includes capturing options from users, supporting host names, and printing statistics.

%%[substart] 

### **📚 Knowledge Pre-requisites**

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Knowledge in DNS is helpful.
    
* [ESP Embedded Rust: Ping CLI Part 1](https://apollolabsblog.hashnode.dev/esp-embedded-rust-ping-cli-app-part-1)
    

### **💾 Software Setup**

All the code presented in this post is available on the apollolabs [ESP32C3 git repo](https://github.com/apollolabsdev/ESP32C3). Note that if the code on the git repo is slightly different, it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [**here**](https://wokwi.com/projects/389975860067518465).

### **🛠 Hardware Setup**

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp&auto=compress,format&format=webp align="center")
    

## **👨‍🎨** Software Design

I'm going to make a small modification to the program options to ease the development a bit. Here's the modified version of the `help`:

```rust
Ping is a utility that sends ICMP Echo Request packets to a specified network host 
(either identified by its IP address or hostname) to test connectivity and measure round-trip time.

Usage: ping [options] <hostname/IP>

Options:
  --count=<number>     Number of ICMP Echo Request packets to send (default is 4).
  --interval=<seconds> Set the interval between successive ping packets in seconds.
  --timeout=<seconds>  Specify a timeout value for each ping attempt.
  --size=<bytes>       Set the size of the ICMP packets.

Examples:
  ping 192.168.1.1          # Ping the IP address 192.168.1.1
  ping example.com          # Ping the hostname 'example.com'
  ping --count=10 google.com     # Send 10 ping requests to google.com
  ping --interval=0.5 --size=100 example.com  # Ping with interval of 0.5 seconds and packet size of 100 bytes to 'example.com'
```

In last week's code there were three steps:

1. Setup CLI Root Menu & Callback
    
2. CLI Start-Up
    
3. Create Ping App Logic
    

Here are the areas that need to be updated:

* The root menu in step 1 needs to be updated with items to incorporate the command options.
    
* The app logic in step 3 needs to incorporate the logic to acquire the optional parameters, process the parameters, and adjust the printed output.
    

## **👨‍💻** Code Implementation

### **📥 Crate Imports**

In this implementation the crates required are as follows:

* The `esp_idf_hal` crate to import the needed device hardware abstractions.
    
* `menu` to provide the menu CLI structs and abstractions.
    
* `std::fmt` to import the `Write` trait.
    

```rust
use esp_idf_hal::delay::BLOCK;
use esp_idf_hal::gpio;
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_hal::prelude::*;
use esp_idf_hal::uart::*;
use menu::*;
use std::fmt::Write;
```

### 📝 Updating the Root Menu

We need to update the `ROOT_MENU` ping command `Item` to accommodate the options shown in the help output earlier. This includes `count`, `interval`, `timeout`, and `size`. In the parameter list, these all need to be of type `Parameter::NamedValue` since they are all named parameters with an argument ([link to menu crate documentation](https://docs.rs/menu/latest/menu/enum.Parameter.html)). Here's the updated `Item` :

```rust
&Item {
    item_type: ItemType::Callback {
        function: ping_app,
        parameters: &[Parameter::Mandatory {
            parameter_name: "hostname/IP",
            help: Some("IP address or hostname"),
        },
        Parameter::NamedValue {
            parameter_name: "count",
            argument_name: "cnt",
            help: Some("Packet count"),
        },
        Parameter::NamedValue {
            parameter_name: "interval",
            argument_name: "int",
            help: Some("Interval between counts"),
        },
        Parameter::NamedValue {
            parameter_name: "timeout",
            argument_name: "to",
            help: Some("timeout for each ping attempt"),
        },
        Parameter::NamedValue {
            parameter_name: "size",
            argument_name: "sz",
            help: Some("Set the size of the packet"),
        },],
    },
    command: "ping",
    help: Some("
    Ping is a utility that sends ICMP Echo Request packets to a specified network host 
    (either identified by its IP address or hostname) to test connectivity and measure round-trip time.

    Usage: ping [options] <hostname/IP>
    
    Options:
      --count=<number>     Number of ICMP Echo Request packets to send (default is 4).
      --interval=<seconds> Set the interval between successive ping packets in seconds.
      --timeout=<seconds>  Specify a timeout value for each ping attempt.
      --size=<bytes>       Set the size of the ICMP packets.
      --help               Display this help message and exit.
    
    Examples:
      ping 192.168.1.1          # Ping the IP address 192.168.1.1
      ping example.com          # Ping the hostname 'example.com'
      ping -count=10 google.com     # Send 10 ping requests to google.com
      ping -interval=0.5 -size=100 example.com  # Ping with interval of 0.5 seconds and packet size of 100 bytes to 'example.com'
    "),
},
```

### 👨‍💻 Acquire and Process Parameters

**🌐 Acquiring and Processing a Hostname**

In the [last post](https://apollolabsblog.hashnode.dev/esp-embedded-rust-ping-cli-app-part-1), in the ping app CLI `ping_app` callback function, we supported pinging only raw IP addresses using the `Ipv4Addrfrom_str` method. Meaning if we wanted to ping a hostname like www.google.com, we wouldn't be able to. To support hostnames, we'd need an abstraction that can parse the acquired string and resolve the hostname to an IP. `std::net::ToSocketAddrs` provides such abstractions. Heres the updated code:

```rust
// Read terminal input
let ip_str = argument_finder(item, args, "hostname/IP").unwrap().unwrap();

// Resolve IP Address
let addresses = (ip_str, 0)
    .to_socket_addrs()
    .expect("Unable to resolve domain")
    .next()
    .unwrap();

// Convert to IP v4 type address
let addr = match addresses {
    std::net::SocketAddr::V4(a) => *a.ip(),
    std::net::SocketAddr::V6(_) => {
        writeln!(context, "Address not compatible, try again").unwrap();
        return;
    }
};
```

I'm going to broadly explain what is happening here and would recommend referring to the `std::net::ToSocketAddrs` documentation for detail. The first part of the code is similar to before where we read the terminal input. In the second part, `to_socket_addrs` is being used to parse and resolve an address from the user entry. `to_socket_addrs` returns a collection of possible addresses in which only the first one is being used.

The addresses returned by `to_socket_addrs` are of IpAddr type that could be either a IPv4 or IPv6 address. As such, the second part of the code makes sure that its an IPv4 address that is being used since thats what `EspPing` supports.

**📝 Acquiring and Processing Options**

We earlier added four command options the user can enter. These options would need to update the `ping_configEspPing` configuration struct. As such, a `default` instance of `ping_config` is created first such that if the user doesn't enter any options to update a member then we opt for a `default` configuration. Here's the code:

```rust
// Setup Default Ping Config
let mut ping_config = PingConfiguration::default();

// Obtain CLI Options and Modify Default Configuration Accordingly
ping_config.count = 1;
let mut ping_attempts = 4;

match argument_finder(item, args, "count") {
    Ok(arg) => match arg {
        Some(cnt) => ping_attempts = FromStr::from_str(cnt).unwrap(),
        None => (),
    },
    Err(_) => (),
}

match argument_finder(item, args, "interval") {
    Ok(arg) => match arg {
        Some(inter) => {
            ping_config.interval = Duration::from_secs(FromStr::from_str(inter).unwrap())
        }
        None => (),
    },
    Err(_) => (),
}

match argument_finder(item, args, "timeout") {
    Ok(arg) => match arg {
        Some(to) => ping_config.timeout = Duration::from_secs(FromStr::from_str(to).unwrap()),
        None => (),
    },
    Err(_) => (),
}

match argument_finder(item, args, "size") {
    Ok(arg) => match arg {
        Some(sz) => ping_config.data_size = FromStr::from_str(sz).unwrap(),
        None => (),
    },
    Err(_) => (),
}
```

Note how `ping_config.count` is set to 1. This is because if it's anything other than 1 then it would send multiple `byte_size` packets per attempt. Additionally, we'd want to print each `ping` attempt individually a number of times that the user defines. For that, the `ping_attempts` variable is introduced with a default of `4`.

> 📝 **Note:**`count` is used in two different contexts. `count` that is provided as a user option is essentially what we are calling the number of ping attempts. On the other hand, while `ping_config.count` also refers to the number of attempts, it refers to attempts done in one ping operation. This means we wouldn't be able to print every attempt individually. To circumvent this, `ping_config.count` is always set to 1 and `ping_attempts` controls how many times an `EspPing` is performed.

### 👨‍💻 Adjust the Printed Output

In the first part of the printed output, we need to adjust it to reflect the input the user provided. Additionally, we need to introduce some variables to collect information for printing the statistics later. These are `summary`, `times`, and `rx_count`. Here is the adjusted output:

```rust
// Update CLI
// Pinging {IP} with {x} bytes of data
writeln!(
    context,
    "Pinging {} [{:?}] with {} bytes of data\n",
    ip_str, addr, ping_config.data_size
)
.unwrap();

let mut summary = Summary::default();
let mut times: Vec<u128> = Vec::new();
let mut rx_count = 0;

// Ping 4 times and print results in following format:
// Reply from {IP}: bytes={summary.recieved} time={summary.time} TTL={summary.timeout}
for _n in 1..=ping_attempts {
    summary = ping.ping(addr, &ping_config).unwrap();

    writeln!(
        context,
        "Reply from {:?}: bytes = {}, time = {:?}, TTL = {:?}",
        addr, ping_config.data_size, summary.time, ping_config.timeout
    )
    .unwrap();

    // Update values for statistics
    times.push(summary.time.as_millis());
    if summary.transmitted == summary.received {
        rx_count += 1;
    }
}
```

Afterward, the ping stats are appended with the following commands:

```rust
// Print ping statistics in following format:
// Ping statistics for {IP}:
//      Packets: Sent = {sent}, Received  = {rec}, Lost = {loss} <{per}% Loss>
// Approximate round trip times in milliseconds:
//      Minimum = {min}ms, Maximum = {max}ms, Average = {avg}ms
writeln!(context, "\nPing Statistics for {:?}", addr).unwrap();
writeln!(
    context,
    "     Packets: Sent = {}, Received  = {}, Lost = {} <{}% loss>",
    ping_attempts,
    rx_count,
    ping_attempts - rx_count,
    ((ping_attempts - rx_count) / ping_attempts) * 100
)
.unwrap();
writeln!(context, "Approximate round trip times in milliseconds:").unwrap();
writeln!(
    context,
    "     Minimum = {} ms, Maximum = {} ms, Average = {} ms",
    times.iter().min().unwrap(),
    times.iter().max().unwrap(),
    times.iter().sum::<u128>() / times.len() as u128,
)
.unwrap();
```

That's it for code!

## 📱Full Application Code

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Also, the Wokwi project can be accessed [**here**](https://wokwi.com/projects/389975860067518465).

```rust
use esp_idf_hal::delay::BLOCK;
use esp_idf_hal::gpio;
use esp_idf_hal::prelude::*;
use esp_idf_hal::uart::*;
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::ping::{Configuration as PingConfiguration, EspPing, Summary};
use esp_idf_svc::wifi::{AuthMethod, BlockingWifi, ClientConfiguration, Configuration, EspWifi};
use menu::*;
use std::fmt::Write;
use std::net::ToSocketAddrs;
use std::str::FromStr;
use std::time::Duration;

// CLI Root Menu Struct Initialization
const ROOT_MENU: Menu<UartDriver> = Menu {
    label: "root",
    items: &[
        &Item {
            item_type: ItemType::Callback {
                function: hello_name,
                parameters: &[Parameter::Mandatory {
                    parameter_name: "name",
                    help: Some("Enter your name"),
                }],
            },
            command: "hw",
            help: Some("This is the help for the hello, name hw command!"),
        },
        &Item {
            item_type: ItemType::Callback {
                function: ping_app,
                parameters: &[Parameter::Mandatory {
                    parameter_name: "hostname/IP",
                    help: Some("IP address or hostname"),
                },
                Parameter::NamedValue {
                    parameter_name: "count",
                    argument_name: "cnt",
                    help: Some("Packet count"),
                },
                Parameter::NamedValue {
                    parameter_name: "interval",
                    argument_name: "int",
                    help: Some("Interval between counts"),
                },
                Parameter::NamedValue {
                    parameter_name: "timeout",
                    argument_name: "to",
                    help: Some("timeout for each ping attempt"),
                },
                Parameter::NamedValue {
                    parameter_name: "size",
                    argument_name: "sz",
                    help: Some("Set the size of the packet"),
                },],
            },
            command: "ping",
            help: Some("
            Ping is a utility that sends ICMP Echo Request packets to a specified network host 
            (either identified by its IP address or hostname) to test connectivity and measure round-trip time.

            Usage: ping [options] <hostname/IP>
            
            Options:
              --count=<number>     Number of ICMP Echo Request packets to send (default is 4).
              --interval=<seconds> Set the interval between successive ping packets in seconds.
              --timeout=<seconds>  Specify a timeout value for each ping attempt.
              --size=<bytes>       Set the size of the ICMP packets.
              --help               Display this help message and exit.
            
            Examples:
              ping 192.168.1.1          # Ping the IP address 192.168.1.1
              ping example.com          # Ping the hostname 'example.com'
              ping -count=10 google.com     # Send 10 ping requests to google.com
              ping -interval=0.5 -size=100 example.com  # Ping with interval of 0.5 seconds and packet size of 100 bytes to 'example.com'
            "),
        },
    ],
    entry: None,
    exit: None,
};

fn main() -> anyhow::Result<()> {
    // Take Peripherals
    let peripherals = Peripherals::take().unwrap();
    let sysloop = EspSystemEventLoop::take()?;
    let nvs = EspDefaultNvsPartition::take()?;

    let mut wifi = BlockingWifi::wrap(
        EspWifi::new(peripherals.modem, sysloop.clone(), Some(nvs))?,
        sysloop,
    )?;

    wifi.set_configuration(&Configuration::Client(ClientConfiguration {
        ssid: "Wokwi-GUEST".try_into().unwrap(),
        bssid: None,
        auth_method: AuthMethod::None,
        password: "".try_into().unwrap(),
        channel: None,
    }))?;

    // Start Wifi
    wifi.start()?;

    // Connect Wifi
    wifi.connect()?;

    // Wait until the network interface is up
    wifi.wait_netif_up()?;

    println!("Wifi Connected");

    // Configure UART
    // Create handle for UART config struct
    let config = config::Config::default().baudrate(Hertz(115_200));

    // Instantiate UART
    let mut uart = UartDriver::new(
        peripherals.uart0,
        peripherals.pins.gpio21,
        peripherals.pins.gpio20,
        Option::<gpio::Gpio0>::None,
        Option::<gpio::Gpio1>::None,
        &config,
    )
    .unwrap();

    // This line is for Wokwi only so that the console output is formatted correctly
    uart.write_str("\x1b[20h").unwrap();

    // Create a buffer to store CLI input
    let mut clibuf = [0u8; 64];
    // Instantiate CLI runner with root menu, buffer, and uart
    let mut r = Runner::new(ROOT_MENU, &mut clibuf, uart);

    loop {
        // Create single element buffer for UART characters
        let mut buf = [0_u8; 1];
        // Read single byte from UART
        r.context.read(&mut buf, BLOCK).unwrap();
        // Pass read byte to CLI runner for processing
        r.input_byte(buf[0]);
    }
}

// Callback function for hw command
fn hello_name<'a>(
    _menu: &Menu<UartDriver>,
    item: &Item<UartDriver>,
    args: &[&str],
    context: &mut UartDriver,
) {
    // Print to console passed "name" argument
    writeln!(
        context,
        "Hello, {}!",
        argument_finder(item, args, "name").unwrap().unwrap()
    )
    .unwrap();
}

// Callback function for ping command
fn ping_app<'a>(
    _menu: &Menu<UartDriver>,
    item: &Item<UartDriver>,
    args: &[&str],
    context: &mut UartDriver,
) {
    // Retreieve CLI Input
    let ip_str = argument_finder(item, args, "hostname/IP").unwrap().unwrap();

    // Resolve IP Address
    let addresses = (ip_str, 0)
        .to_socket_addrs()
        .expect("Unable to resolve domain")
        .next()
        .unwrap();

    // Convert to IP v4 type address
    let addr = match addresses {
        std::net::SocketAddr::V4(a) => *a.ip(),
        std::net::SocketAddr::V6(_) => {
            writeln!(context, "Address not compatible, try again").unwrap();
            return;
        }
    };

    // Create EspPing instance
    let mut ping = EspPing::new(0_u32);

    // Setup Default Ping Config
    let mut ping_config = PingConfiguration::default();

    // Obtain CLI Options and Modify Default Configuration Accordingly
    ping_config.count = 1;
    let mut ping_attempts = 4;

    match argument_finder(item, args, "count") {
        Ok(arg) => match arg {
            Some(cnt) => ping_attempts = FromStr::from_str(cnt).unwrap(),
            None => (),
        },
        Err(_) => (),
    }

    match argument_finder(item, args, "interval") {
        Ok(arg) => match arg {
            Some(inter) => {
                ping_config.interval = Duration::from_secs(FromStr::from_str(inter).unwrap())
            }
            None => (),
        },
        Err(_) => (),
    }

    match argument_finder(item, args, "timeout") {
        Ok(arg) => match arg {
            Some(to) => ping_config.timeout = Duration::from_secs(FromStr::from_str(to).unwrap()),
            None => (),
        },
        Err(_) => (),
    }

    match argument_finder(item, args, "size") {
        Ok(arg) => match arg {
            Some(sz) => ping_config.data_size = FromStr::from_str(sz).unwrap(),
            None => (),
        },
        Err(_) => (),
    }

    println!("{:?}", ping_config);

    // Update CLI
    // Pinging {IP} with {x} bytes of data
    writeln!(
        context,
        "Pinging {} [{:?}] with {} bytes of data\n",
        ip_str, addr, ping_config.data_size
    )
    .unwrap();

    let mut summary = Summary::default();
    let mut times: Vec<u128> = Vec::new();
    let mut rx_count = 0;

    // Ping 4 times and print results in following format:
    // Reply from {IP}: bytes={summary.recieved} time={summary.time} TTL={summary.timeout}
    for _n in 1..=ping_attempts {
        summary = ping.ping(addr, &ping_config).unwrap();

        writeln!(
            context,
            "Reply from {:?}: bytes = {}, time = {:?}, TTL = {:?}",
            addr, ping_config.data_size, summary.time, ping_config.timeout
        )
        .unwrap();

        // Update values for statistics
        times.push(summary.time.as_millis());
        if summary.transmitted == summary.received {
            rx_count += 1;
        }
    }

    // Print ping statstics in following format:
    // Ping statistics for {IP}:
    //      Packets: Sent = {sent}, Recieved  = {rec}, Lost = {loss} <{per}% Loss>
    // Approximate round trip times in milliseconds:
    //      Minimum = {min}ms, Maximum = {max}ms, Average = {avg}ms
    writeln!(context, "\nPing Statistics for {:?}", addr).unwrap();
    writeln!(
        context,
        "     Packets: Sent = {}, Recieved  = {}, Lost = {} <{}% loss>",
        ping_attempts,
        rx_count,
        ping_attempts - rx_count,
        ((ping_attempts - rx_count) / ping_attempts) * 100
    )
    .unwrap();
    writeln!(context, "Approximate round trip times in milliseconds:").unwrap();
    writeln!(
        context,
        "     Minimum = {} ms, Maximum = {} ms, Average = {} ms",
        times.iter().min().unwrap(),
        times.iter().max().unwrap(),
        times.iter().sum::<u128>() / times.len() as u128,
    )
    .unwrap();
}
```

## **Conclusion**

In this post, a ping CLI application replica was built on an ESP32C3 using Rust and the supporting `std` library crates. The current version is an updated version of a CLI presented in a previous post. In the application in this post, hostnames and options are supported. Additionally, ping statistics are reported as part of the output. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]