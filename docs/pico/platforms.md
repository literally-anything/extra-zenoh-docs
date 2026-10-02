# Supported platforms (RTOS)

zenoh-pico runs on desktop OSes, several RTOSes and bare-metal frameworks. Each target is a **platform
profile**, a CMake file in `cmake/platforms/<name>.cmake` that lists the system-layer and link sources to
compile. You pick one with `-DZP_PLATFORM=<name>`. Host builds detect it automatically: Linux → `linux`,
macOS → `macos`, *BSD → `bsd`, Windows → `windows`, Emscripten → `emscripten`, Raspberry Pi Pico SDK →
`rpi_pico`. A `Generic` (bare-metal) CMake system **must** set `ZP_PLATFORM` explicitly.

## Platform matrix

Links are what each profile compiles in at 1.10.1 (from `cmake/platforms/*.cmake`). Each one still has to be
turned on with its `Z_FEATURE_LINK_*` flag ([Capabilities](capabilities.md#feature-flags)).

| Profile | OS / RTOS | Threads (`Z_FEATURE_MULTI_THREAD=1`) | TCP | UDP unicast | UDP multicast | Serial | Other |
|---|---|---|---|---|---|---|---|
| `linux` | Linux | pthreads | ✅ | ✅ | ✅ | ✅ tty | raw Ethernet, TLS |
| `macos` | macOS | pthreads | ✅ | ✅ | ✅ | ✅ tty | TLS |
| `bsd` | FreeBSD/NetBSD/OpenBSD | pthreads | ✅ | ✅ | ✅ | ✅ tty | TLS |
| `posix_compatible` | Other POSIX systems (`-DPOSIX_COMPATIBLE=ON`) | pthreads | ✅ | ✅ | ✅ | ✅ tty | |
| `windows` | Windows (Winsock2) | Win32 threads, SRW locks | ✅ | ✅ | ✅ | ❌ | |
| `zephyr` | Zephyr RTOS | Zephyr pthreads | ✅ | ✅ | ✅ | ✅ (device name) | |
| `freertos_lwip` | FreeRTOS + lwIP | FreeRTOS tasks | ✅ | ✅ | ✅ | ❌ | |
| `freertos_plus_tcp` | FreeRTOS + FreeRTOS-Plus-TCP | FreeRTOS tasks | ✅ | ✅ | **❌ (forced off)** | ❌ | |
| `rpi_pico` | Raspberry Pi Pico SDK + FreeRTOS + lwIP (Pico W) | FreeRTOS tasks | ✅ | ✅ | ✅ | ✅ UART, **USB CDC** | |
| `espidf` | ESP-IDF (ESP32 family) | FreeRTOS tasks, pthread mutexes | ✅ | ✅ | ✅ | ✅ `UART_0..2` or pins | |
| `arduino_esp32` | Arduino on ESP32 | FreeRTOS tasks, pthread mutexes | ✅ | ✅ | ✅ | ✅ `UART_0..2` or pins | **Bluetooth** (SPP) |
| `mbed` | Mbed OS | `rtos::Thread` | ✅ | ✅ | ✅ | ✅ (pins only) | |
| `opencr` | Arduino on ROBOTIS OpenCR | **❌, must build with `Z_FEATURE_MULTI_THREAD=0`** | ✅ | ✅ | ✅ | ❌ | |
| `threadx_stm32` | Azure RTOS ThreadX on STM32 (CubeMX HAL) | ThreadX threads | ❌ | ❌ | ❌ | ✅ (board UART only) | |
| `flipper` | Flipper Zero firmware | FuriThread | ❌ | ❌ | ❌ | ✅ `usart` / `lpuart` | |
| `emscripten` | Browser / WebAssembly | pthreads | ❌ | ❌ | ❌ | ❌ | **WebSocket** only |

Details that the table can't show:

- **TLS** needs mbedtls (2.x or 3.x, found with `pkg-config`) and uses POSIX socket headers. Only the Unix
  platform header defines a TLS socket, so at 1.10.1 TLS is effectively **Linux/macOS/BSD only**.
- **WebSocket** (`Z_FEATURE_LINK_WS`) is rejected at configure time on anything but `emscripten`:
  `Z_FEATURE_LINK_WS is currently only supported on the emscripten platform.`
- **Raw Ethernet** uses `AF_PACKET` and is compiled only on Linux (`#if defined(__linux)`).
- **Bluetooth** exists only for Arduino-ESP32 (`bt_arduino_esp32.cpp`, Bluetooth Classic SPP).
- **OpenCR**: the thread, mutex and condition-variable functions are stubs that return `-1`.
- **FreeRTOS** (`freertos_lwip`, `freertos_plus_tcp`, `rpi_pico`): `z_malloc` uses `pvPortMalloc`, and
  `z_realloc` isn't implemented (returns `NULL`).
- **Serial device naming** differs per platform: see [Transports](transports.md#serial).
- `opencr` and `posix_compatible` set `CHECK_THREADS OFF`, so CMake doesn't look for a thread library.
  `arduino_esp32`, `espidf`, `mbed`, `rpi_pico`, `threadx_stm32` and `flipper` do the same.

## Boards the README says were tested

| Platform | Boards |
|---|---|
| Zephyr | reel_board, nucleo-f767zi, nucleo-f420zi, nRF52840 |
| Arduino ESP32, ESP-IDF | az-delivery-devkit-v4 (ESP32) |
| Mbed OS | nucleo-f747zi, nucleo-f429zi |
| OpenCR | ROBOTIS OpenCR 1.0 |
| Raspberry Pi Pico | Pico W, Pico 2 W |

## Building for each platform

### Linux, macOS, BSD, Windows

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release [-DZ_FEATURE_…=…]
cmake --build build -j && cmake --install build
```

`make` (`GNUmakefile`) wraps the same and accepts `BUILD_TYPE`, `ZENOH_LOG`, `BUILD_EXAMPLES`,
`BUILD_TESTING` and the feature variables. Debian/RPM packages of the library are published on
download.eclipse.org.

!!! warning "Use `Release`, not `MinSizeRel`"
    `CMakeLists.txt` adds its optimisation flags only for `Release`. Any other build type, including
    `MinSizeRel` and `RelWithDebInfo`, falls into the debug branch and appends `-g -O0 -Werror …` after
    CMake's own flags, so a `MinSizeRel` build ends up compiled with **`-O0`** (seen in
    `compile_commands.json`). To optimise for size, use `Release` and override the compiler flags yourself.

### PlatformIO (Arduino ESP32, ESP-IDF, Mbed, Zephyr, OpenCR)

`library.json` declares zenoh-pico as a PlatformIO library for the `arduino`, `espidf`, `mbed` and `zephyr`
frameworks. `extra_script.py` picks the defines from the framework and board: `ZENOH_ARDUINO_ESP32`,
`ZENOH_ARDUINO_OPENCR` (board `opencr`), `ZENOH_ESPIDF`, `ZENOH_MBED`, `ZENOH_ZEPHYR`, or `ZENOH_GENERIC`
when the build environment sets `ZENOH_GENERIC=1`.

```ini
; platformio.ini
lib_deps = https://github.com/eclipse-zenoh/zenoh-pico
```

### Raspberry Pi Pico (W / 2 W)

Needs the Pico SDK and the FreeRTOS kernel (`PICO_SDK_PATH`, `FREERTOS_KERNEL_PATH`). The examples take
`PICO_BOARD` (`pico`, `pico_w`, `pico2`, `pico2_w`), `WIFI_SSID`, `WIFI_PASSWORD`, `ZENOH_CONFIG_MODE`,
`ZENOH_CONFIG_CONNECT` and `ZENOH_CONFIG_LISTEN`:

```bash
cd examples/rpi_pico
cmake -Bbuild -DPICO_BOARD=pico_w -DWIFI_SSID=… -DWIFI_PASSWORD=… \
      -DZENOH_CONFIG_MODE=client -DZENOH_CONFIG_CONNECT=tcp/192.168.1.10:7447
cmake --build build      # flash the .uf2
```

### STM32 + ThreadX (serial only)

Integrated by hand into an STM32CubeIDE project (README §2.2.7): enable the AZURE-RTOS middleware and a UART
with circular RX DMA, add `src/` and `include/`, define `ZENOH_THREADX_STM32` and `ZENOH_HUART=huartN`, and
make the ThreadX byte pool **larger than 25 kB**. The host side is a `zenohd` built with
`transport_serial`: `zenohd -l serial//dev/ttyACM0#baudrate=115200`.

### FreeRTOS + FreeRTOS-Plus-TCP

`examples/freertos_plus_tcp/` has a CMake project, including single-thread variants (`z_pub_st`, `z_sub_st`).

### Zephyr module

`zephyr/module.yml`, `zephyr/CMakeLists.txt` and `zephyr/Kconfig.zenoh` let you add zenoh-pico as a Zephyr
module. The Kconfig options are `CONFIG_ZENOH_PICO` and `CONFIG_ZENOH_PICO_{LINK_SERIAL, MULTI_THREAD,
PUBLICATION, SUBSCRIPTION, QUERY, QUERYABLE, RAWETH_TRANSPORT, LINK_TCP, LINK_UDP_UNICAST,
LINK_UDP_MULTICAST, SCOUTING, LINK_WS}`. Each one maps to `Z_FEATURE_*=0/1`. These `bool` options have **no
defaults**, so everything is off until you enable it in `prj.conf`. `CONFIG_ZENOH_PICO_THREADS_NUM` (default
4) sets how many pthread stacks are preallocated.

## Porting to a new platform

1. Write a profile (start from `examples/platforms/myplatform.cmake`):
   ```cmake
   set(ZP_PLATFORM_COMPILE_DEFINITIONS ZENOH_MYRTOS)
   set(ZP_PLATFORM_SOURCE_FILES
       "${PROJECT_SOURCE_DIR}/src/system/myrtos/system.c"      # memory, time, random, threads
       "${PROJECT_SOURCE_DIR}/src/system/socket/lwip.c"        # reuse lwIP glue if you have lwIP
       "${PROJECT_SOURCE_DIR}/src/link/transport/tcp/tcp_lwip.c"
       "${PROJECT_SOURCE_DIR}/src/link/transport/udp/udp_lwip.c"
       "${PROJECT_SOURCE_DIR}/src/link/transport/serial/uart_myrtos.c")
   if(ZP_UDP_MULTICAST_ENABLED)
     list(APPEND ZP_PLATFORM_SOURCE_FILES
          "${PROJECT_SOURCE_DIR}/src/link/transport/udp/udp_multicast_lwip.c"
          "${PROJECT_SOURCE_DIR}/src/link/transport/udp/udp_multicast_lwip_common.c")
   endif()
   ```
   Optional variables: `ZP_PLATFORM_INCLUDE_DIRS`, `ZP_PLATFORM_COMPILE_OPTIONS`,
   `ZP_PLATFORM_LINK_LIBRARIES`, `ZP_PLATFORM_SYSTEM_LAYER` (when the system-layer name differs from the
   profile name) and `ZP_PLATFORM_SYSTEM_PLATFORM_HEADER` (to use your own platform header instead of a
   built-in one).
2. Implement the platform API from `include/zenoh-pico/system/common/platform.h` (memory, random, clock,
   sleep, and with multi-thread also tasks, mutexes and condition variables) and the per-link functions
   for the links you need ([architecture](architecture.md#the-platform-layer)). If your stack is lwIP,
   the `*_lwip.c` sources do most of the work.
3. Build with `-DZP_PLATFORM=myrtos`. To keep the profile **out of tree**, ship it in a CMake package and
   load it with `-DZP_EXTERNAL_PACKAGES=<pkg>`. The package's config file calls
   `zp_add_platform_dir(<dir>)`. `examples/packages/zenohpico-mylinux/` is a complete example.

The old `ZP_SYSTEM_LAYER` variable is gone: setting it is a configure error (`ZP_SYSTEM_LAYER is no longer
supported. Use ZP_PLATFORM instead.`).

## Sources

- `zenoh-pico@1.10.1`: `cmake/platforms/*.cmake`, `cmake/platforms.cmake` (default detection),
  `CMakeLists.txt` (checks, TLS/mbedtls, build types), `docs/platforms.rst`, `README.md` (§2)
- `include/zenoh-pico/system/platform/*.h` (thread and mutex types), `src/system/*/system.c`
- `src/link/transport/*` (which links exist for which platform)
- `library.json`, `extra_script.py` (PlatformIO), `zephyr/` (module), `examples/*/`
