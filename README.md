# USB Monitor using Netlink

USB Monitor is desktop application for Linux, developed in C++ and Qt, providing real-time tracking of USB device connections and disconnections. It's designed to be highly efficient and deliver detailed information.

The key to its uniqueness is the use of **Netlink sockets**. This offers a fast, direct communication channel between the Linux kernel and user-space applications. Thanks to Netlink, USB Monitor gets immediate notifications about device events as soon as they occur. This is far more efficient than older methods that relied on constantly scanning `dmesg` system logs or `/sys` files.

This project showcases how modern system mechanisms can be leveraged to create convenient and powerful tools that simplify and enhance the Linux experience.

---

### **Table of Contents**

- [Visual Overview](#visual-overview)
- [Project Structure](#project-structure)
- [Features](#features)
- [How It Works](#how-it-works)
- [Technical Dependencies](#technical-dependencies)
- [Monitoring System Device Events via Netlink](#monitoring-system-device-events-via-netlink)
- [Building and Running](#building-and-running)
- [License](#license)

---

## **Visual Overview**

<img width="864" alt="455694081-7178300d-f4b2-4266-ab76-cdeb0e3970ef" src="https://github.com/user-attachments/assets/76ce469b-fe82-41c2-a1df-7810ea6844f5" />

---

## **Project Structure**

```bash
.
├── cmake-build-debug/
│   └── ... 
├── CMakeLists.txt
├── main.cpp
├── README.md
├── usb-monitor.pro
├── usbmonitor.h
├── usbworker.cpp
└── usbworker.h
```
---

## **Features**

- **Real-time Monitoring**  
  Immediate reaction to USB device connect/disconnect events thanks to direct communication with the Linux kernel via Netlink (uevent).

- **Low System Load**  
  Uses a blocking `select()` call instead of constant polling — the processor is engaged only when an actual event occurs.

- **Detailed Information**  
  Displays physical path, device name/manufacturer, and storage characteristics, gathered from various sources.

- **Modern Interface (Qt)**  
  Features a convenient table, a pleasant dark theme, and a console for event logging.

---

## **How It Works**

The project comprises two logically separated components: `USBMonitorGUI` (interface) and `USBWorker` (logic).

### Netlink Socket Creation

Upon startup, `USBWorker` creates an `AF_NETLINK` family socket and subscribes to the `NETLINK_KOBJECT_UEVENT` multicast group. This step turns the application into a passive listener for "uevents"—messages the Linux kernel generates whenever hardware states change (connection, disconnection, modification, etc.).

  ```bash
    netlink_socket = socket(AF_NETLINK, SOCK_RAW, NETLINK_KOBJECT_UEVENT);
  ```

### Event Processing (Udev Events)

The Worker waits in an infinite loop for data using the `select()` function, which efficiently blocks the thread until a new event arrives.

 ```bash
   const int ret = select(netlink_socket + 1, &fds, nullptr, nullptr, &tv);
     if (ret < 0) {
       if (errno == EINTR) continue; 
         perror("select");
         break;
     }
   if (ret == 0) continue;
 ```

Each received message is a null-byte-separated string containing `KEY=VALUE` pairs. This data is parsed into a map for easy access.

  ```bash
    void USBWorker::parseUEvent(const char* buffer, std::map<std::string, std::string>& uevent_data) {
        const char* s = buffer;
        while (*s) {
            std::string line(s);
            if (const size_t equals_pos = line.find('='); equals_pos != std::string::npos) {
                uevent_data[line.substr(0, equals_pos)] = line.substr(equals_pos + 1);
            }
            s += line.length() + 1;
        }
    }
   ```

The application intelligently filters events, focusing only on those with `SUBSYSTEM` set to `usb` or `block`. Why both?

* A `usb` event contains basic connection information (Vendor ID, Product ID).
* A `block` event pertains to block devices and provides detailed storage information associated with the given USB device.

For an `add` event, external commands (`lsusb`, `lsblk`) are executed to translate technical IDs into human-readable names, models, and sizes.

For a `remove` event, the device is simply identified by its `DEVPATH` and removed from the table.

### Safe Communication with GUI

Since `USBWorker` runs in a separate thread, it cannot directly manipulate GUI elements. The well-established **Qt signals and slots** mechanism is employed for communication:

1.  When `USBWorker` processes an event, it emits a signal (e.g., `deviceConnected`).
2.  This signal is connected to a slot in the `USBMonitorGUI` class within the main thread.
3.  Qt ensures safe signal delivery between threads, preventing concurrency issues and maintaining application stability.

---

## **Technical Dependencies**

The following components are required to successfully build and run the project:

* **Operating System:** Linux 
* **C++ Compiler:** С++20
* **Qt Framework:** Qt5 
* **Build System:** CMake or qmake

---

## **Monitoring System Device Events via Netlink**

To track device connection/disconnection events in Linux, the **Netlink** mechanism (family `NETLINK_KOBJECT_UEVENT`) is used, generated by the kernel. This project programmatically listens for these events (see `usbworker.cpp`).

For visualization and verification of this process, a study was conducted using `udevadm monitor` and Wireshark with the `nlmon` module.

<img width="861" alt="456067941-3d8c9d4f-1646-4802-8114-e4fa7c1358ba" src="https://github.com/user-attachments/assets/29e2512d-45bb-4d42-923b-32698f1b989e" />

The screenshot presents a dual monitoring setup.
* In the **upper part**, Wireshark with `nlmon0` captures *general* Netlink packets as network traffic. Although Wireshark may not always fully parse *all* nuances of UEVENT messages (due to Netlink protocol specifics and `nlmon` capture peculiarities), it confirms that Netlink packets exist and are transmitted through the kernel.
* In the **lower part**, the terminal with `udevadm monitor` directly displays *target* Netlink UEVENT events in an easy-to-read format. This *clearly confirms* the generation of these events by the kernel upon USB device connection. This C++ program performs a similar process, programmatically listening to these same events.

---
## **Building and Running**

1.  **Clone the repository:**

    ```bash
    git clone https://github.com/xadazhii/netlink-sockets.git
    ```
    ```bash
     cd ./netlink-sockets
    ```

2.  **Install dependencies** (example for Debian/Ubuntu based systems):

    ```bash
    sudo apt update
    sudo apt install build-essential qtbase5-dev qt5-qmake qtchooser
    ```
    
    ```bash
    sudo apt install cmake
    ```
    
4.  **Build the project using `qmake` and `make`:**

    ```bash
    mkdir build
    cd build
    cmake .. 
    make -j$(nproc) 
    ```

5.  **Run the application:** The executable (typically named after your project) will be created in the current directory.

    ```bash
    ./USBdevice
    ```
---

## **License**

This project is owned by **Adazhii Kristina**.
