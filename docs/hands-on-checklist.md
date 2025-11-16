# Hands-On Checklist: Getting Started on Your Raspberry Pi

## Your Current Situation
- You have a Raspberry Pi
- It's currently in CLI mode
- You want to build a compositor from scratch
- Your friend mentioned XFCE (they're wrong - Pi OS uses LXDE + Labwc)

## Part 1: Investigation (Do This First!)

Run these commands on your Raspberry Pi and **record the output**:

```bash
# 1. What OS version?
cat /etc/os-release

# 2. What's installed?
dpkg -l | grep -E "labwc|wayfire|lxde|xfce|openbox" | tee ~/installed_desktop.txt

# 3. Check graphics hardware
ls -la /dev/dri/
lsmod | grep drm
sudo dmesg | grep -i drm | tail -20

# 4. Check if systemd has graphical target
systemctl get-default

# 5. What's your GPU?
vcgencmd version  # Raspberry Pi specific
# OR
lspci | grep VGA

# 6. Check kernel version
uname -a

# 7. See what display server packages are available
apt-cache search wayland | grep compositor
```

Send me this output and I can give you specific next steps.

---

## Part 2: Set Up Development Environment

### Install Essential Build Tools

```bash
# Update system first
sudo apt update
sudo apt upgrade -y

# Install build essentials
sudo apt install -y \
    build-essential \
    git \
    meson \
    ninja-build \
    pkg-config \
    cmake

# Install graphics development libraries
sudo apt install -y \
    libdrm-dev \
    libgbm-dev \
    libegl1-mesa-dev \
    libgles2-mesa-dev \
    libwayland-dev \
    wayland-protocols \
    libinput-dev \
    libxkbcommon-dev \
    libudev-dev \
    libpixman-1-dev

# Install wlroots (optional, for studying examples)
sudo apt install -y libwlroots-dev wlroots-examples

# Install debugging tools
sudo apt install -y \
    gdb \
    valgrind \
    strace

# Install documentation
sudo apt install -y \
    manpages-dev \
    linux-doc
```

### Create Working Directory

```bash
mkdir -p ~/compositor_dev
cd ~/compositor_dev

# Create subdirectories
mkdir -p {experiments,examples,notes}
```

---

## Part 3: First Experiment - Test DRM Access

### Create Your First Program

```bash
cd ~/compositor_dev/experiments
nano test_drm.c
```

Paste this code:

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

int main() {
    printf("=== DRM Device Test ===\n\n");

    // Try to open DRM device
    int fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);
    if (fd < 0) {
        perror("ERROR: Cannot open /dev/dri/card0");
        printf("\nTroubleshooting:\n");
        printf("1. Check if device exists: ls -la /dev/dri/\n");
        printf("2. Check permissions: id\n");
        printf("3. Add yourself to video group: sudo usermod -a -G video $USER\n");
        printf("4. Then logout and login again\n");
        return 1;
    }

    printf("✓ Successfully opened /dev/dri/card0\n\n");

    // Get DRM version
    drmVersion *version = drmGetVersion(fd);
    if (version) {
        printf("DRM Driver Info:\n");
        printf("  Name: %s\n", version->name);
        printf("  Version: %d.%d.%d\n",
               version->version_major,
               version->version_minor,
               version->version_patchlevel);
        printf("  Description: %s\n", version->desc);
        drmFreeVersion(version);
    }

    // Get resources
    drmModeRes *resources = drmModeGetResources(fd);
    if (!resources) {
        fprintf(stderr, "ERROR: Cannot get DRM resources\n");
        close(fd);
        return 1;
    }

    printf("\nDRM Resources:\n");
    printf("  Connectors: %d\n", resources->count_connectors);
    printf("  CRTCs: %d\n", resources->count_crtcs);
    printf("  Encoders: %d\n", resources->count_encoders);
    printf("  Framebuffers: %d\n", resources->count_fbs);

    // Check each connector
    printf("\nConnected Displays:\n");
    int connected_count = 0;
    for (int i = 0; i < resources->count_connectors; i++) {
        drmModeConnector *conn = drmModeGetConnector(fd, resources->connectors[i]);
        if (conn) {
            if (conn->connection == DRM_MODE_CONNECTED) {
                connected_count++;
                printf("  Display %d:\n", connected_count);
                printf("    ID: %d\n", conn->connector_id);
                printf("    Type: %d\n", conn->connector_type);
                printf("    Available modes: %d\n", conn->count_modes);

                if (conn->count_modes > 0) {
                    printf("    Preferred mode: %dx%d @ %dHz\n",
                           conn->modes[0].hdisplay,
                           conn->modes[0].vdisplay,
                           conn->modes[0].vrefresh);
                }
            }
            drmModeFreeConnector(conn);
        }
    }

    if (connected_count == 0) {
        printf("  No displays connected!\n");
    }

    drmModeFreeResources(resources);
    close(fd);

    printf("\n=== Test Complete ===\n");
    printf("\nNext steps:\n");
    printf("1. If this worked, you're ready to write graphics code!\n");
    printf("2. Try the minimal_drm example next\n");

    return 0;
}
```

### Compile and Run

```bash
# Compile
gcc test_drm.c -o test_drm -ldrm

# Check if you're in video group
groups

# If 'video' is not in the list:
sudo usermod -a -G video $USER
# Then logout and login

# Run the test
./test_drm
```

**Expected Output:**
```
=== DRM Device Test ===

✓ Successfully opened /dev/dri/card0

DRM Driver Info:
  Name: vc4
  Version: 1.0.0
  Description: Broadcom VC4 Graphics

DRM Resources:
  Connectors: 2
  CRTCs: 3
  Encoders: 4
  Framebuffers: 0

Connected Displays:
  Display 1:
    ID: 32
    Type: 11
    Available modes: 15
    Preferred mode: 1920x1080 @ 60Hz

=== Test Complete ===
```

If you see this, **you're ready to proceed!**

---

## Part 4: Display Something Real

Save this as `red_screen.c`:

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/mman.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

int main() {
    // Open DRM
    int fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);
    if (fd < 0) {
        perror("Cannot open DRM device");
        return 1;
    }

    // Get resources
    drmModeRes *res = drmModeGetResources(fd);
    if (!res) {
        fprintf(stderr, "Cannot get resources\n");
        return 1;
    }

    // Find first connected connector
    drmModeConnector *conn = NULL;
    int conn_id = -1;
    for (int i = 0; i < res->count_connectors; i++) {
        conn = drmModeGetConnector(fd, res->connectors[i]);
        if (conn->connection == DRM_MODE_CONNECTED) {
            conn_id = conn->connector_id;
            printf("Using connector %d\n", conn_id);
            break;
        }
        drmModeFreeConnector(conn);
        conn = NULL;
    }

    if (!conn) {
        fprintf(stderr, "No connected display found\n");
        return 1;
    }

    // Get mode
    drmModeModeInfo mode = conn->modes[0];
    printf("Mode: %dx%d @ %dHz\n", mode.hdisplay, mode.vdisplay, mode.vrefresh);

    // Find CRTC
    uint32_t crtc_id = res->crtcs[0];

    // Create dumb buffer
    struct drm_mode_create_dumb create = {
        .width = mode.hdisplay,
        .height = mode.vdisplay,
        .bpp = 32,
    };

    if (drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create) < 0) {
        fprintf(stderr, "Cannot create dumb buffer\n");
        return 1;
    }

    printf("Buffer created: %dx%d, pitch=%d, size=%llu\n",
           create.width, create.height, create.pitch, create.size);

    // Create framebuffer
    uint32_t fb_id;
    if (drmModeAddFB(fd, mode.hdisplay, mode.vdisplay, 24, 32,
                     create.pitch, create.handle, &fb_id)) {
        fprintf(stderr, "Cannot create framebuffer\n");
        return 1;
    }

    printf("Framebuffer created: id=%d\n", fb_id);

    // Map buffer
    struct drm_mode_map_dumb map = { .handle = create.handle };
    if (drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &map)) {
        fprintf(stderr, "Cannot map buffer\n");
        return 1;
    }

    uint8_t *fb = mmap(0, create.size, PROT_READ | PROT_WRITE,
                       MAP_SHARED, fd, map.offset);
    if (fb == MAP_FAILED) {
        perror("Cannot mmap");
        return 1;
    }

    printf("Buffer mapped at %p\n", fb);

    // Fill with red
    printf("Filling buffer with red...\n");
    for (uint32_t y = 0; y < mode.vdisplay; y++) {
        for (uint32_t x = 0; x < mode.hdisplay; x++) {
            uint32_t *pixel = (uint32_t *)(fb + y * create.pitch + x * 4);
            *pixel = 0x00FF0000;  // Red in XRGB8888
        }
    }

    // Save old CRTC
    drmModeCrtc *saved_crtc = drmModeGetCrtc(fd, crtc_id);

    // Set mode
    printf("Setting mode...\n");
    if (drmModeSetCrtc(fd, crtc_id, fb_id, 0, 0,
                       &conn_id, 1, &mode)) {
        fprintf(stderr, "Cannot set CRTC\n");
        return 1;
    }

    printf("\n*** RED SCREEN DISPLAYED! ***\n");
    printf("Press Enter to restore display...\n");
    getchar();

    // Restore
    if (saved_crtc) {
        drmModeSetCrtc(fd, saved_crtc->crtc_id, saved_crtc->buffer_id,
                       saved_crtc->x, saved_crtc->y,
                       &conn_id, 1, &saved_crtc->mode);
        drmModeFreeCrtc(saved_crtc);
    }

    // Cleanup
    munmap(fb, create.size);
    drmModeRmFB(fd, fb_id);
    struct drm_mode_destroy_dumb destroy = { .handle = create.handle };
    drmIoctl(fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
    drmModeFreeConnector(conn);
    drmModeFreeResources(res);
    close(fd);

    printf("Done!\n");
    return 0;
}
```

**Compile and run:**
```bash
gcc red_screen.c -o red_screen -ldrm

# Must run from a VT (Ctrl+Alt+F2), not from a GUI
sudo ./red_screen
```

**You should see a RED screen!** This proves you can directly control the display hardware.

---

## Part 5: Study Existing Compositors

### Clone and Study wlroots Examples

```bash
cd ~/compositor_dev/examples

# Clone wlroots
git clone https://gitlab.freedesktop.org/wlroots/wlroots.git
cd wlroots

# Build it
meson build
ninja -C build

# Study the tinywl example
cd tinywl
cat tinywl.c

# Try to build and run it
meson build
ninja -C build
```

### Study Labwc (What Pi OS Actually Uses)

```bash
cd ~/compositor_dev/examples

# Clone Labwc
git clone https://github.com/labwc/labwc.git
cd labwc

# Read the code
less src/main.c
less src/view.c
```

---

## Part 6: Start Your Own Compositor

Create a project structure:

```bash
mkdir -p ~/compositor_dev/my_compositor
cd ~/compositor_dev/my_compositor

mkdir -p {src,include,build}

cat > src/main.c << 'EOF'
#include <stdio.h>
#include <wayland-server-core.h>

int main(int argc, char *argv[]) {
    printf("My Compositor Starting...\n");

    // Create Wayland display
    struct wl_display *display = wl_display_create();
    if (!display) {
        fprintf(stderr, "Failed to create display\n");
        return 1;
    }

    // Add socket
    const char *socket = wl_display_add_socket_auto(display);
    if (!socket) {
        fprintf(stderr, "Failed to add socket\n");
        return 1;
    }

    printf("Running on WAYLAND_DISPLAY=%s\n", socket);

    // TODO: Add backend, renderer, protocols, etc.

    printf("Compositor would run here...\n");

    // Cleanup
    wl_display_destroy(display);

    return 0;
}
EOF

# Create Makefile
cat > Makefile << 'EOF'
CC = gcc
CFLAGS = $(shell pkg-config --cflags wayland-server)
LIBS = $(shell pkg-config --libs wayland-server)

my_compositor: src/main.c
	$(CC) $(CFLAGS) src/main.c -o my_compositor $(LIBS)

clean:
	rm -f my_compositor
EOF

# Build it
make
```

---

## Troubleshooting Guide

### Problem: "Cannot open /dev/dri/card0"

**Solution:**
```bash
# Check device exists
ls -la /dev/dri/

# Check permissions
id

# Add to video group
sudo usermod -a -G video $USER

# Logout and login
```

### Problem: "No display connected"

**Solution:**
```bash
# Make sure monitor is plugged in
# Check with:
sudo dmesg | grep -i hdmi

# Or for DSI displays:
sudo dmesg | grep -i dsi
```

### Problem: Compiler errors

**Solution:**
```bash
# Make sure all dev packages are installed
sudo apt install libdrm-dev libwayland-dev

# Check pkg-config
pkg-config --cflags --libs libdrm
pkg-config --cflags --libs wayland-server
```

---

## Learning Resources (In Priority Order)

1. **Start Here**: Read the docs I created in `docs/`
2. **DRM Examples**: https://github.com/dvdhrm/docs/tree/master/drm-howto
3. **Wayland Book**: https://wayland-book.com/
4. **wlroots tinywl**: Best minimal example
5. **Labwc source**: Real production compositor

---

## Your 30-Day Plan

### Week 1: Fundamentals
- [ ] Run test_drm.c successfully
- [ ] Display red_screen.c
- [ ] Modify colors (try blue, green, patterns)
- [ ] Read all DRM documentation

### Week 2: Rendering
- [ ] Set up EGL context
- [ ] Render with OpenGL
- [ ] Draw simple shapes
- [ ] Implement double buffering

### Week 3: Input
- [ ] Set up libinput
- [ ] Handle keyboard events
- [ ] Handle mouse movement
- [ ] Draw cursor

### Week 4: Wayland Basics
- [ ] Create Wayland display
- [ ] Accept client connections
- [ ] Handle wl_surface protocol
- [ ] Display client content

After this, you'll have the fundamentals and can build anything!

---

## Quick Reference Commands

```bash
# Check what's using DRM
sudo lsof /dev/dri/card0

# Kill any compositor
sudo pkill labwc
sudo pkill wayfire

# Switch to VT
Ctrl+Alt+F2

# Back to GUI
Ctrl+Alt+F1 (or F7)

# Check Wayland socket
ls -la $XDG_RUNTIME_DIR/

# Run app on specific Wayland display
WAYLAND_DISPLAY=wayland-1 your_app

# Debug DRM
sudo cat /sys/kernel/debug/dri/0/state

# Monitor kernel logs
sudo dmesg -w | grep drm
```

---

Start with `test_drm.c` and work your way through. Once you get the red screen working, you'll have proven you can control the hardware directly. Everything else builds on that foundation!

Let me know which step you're on and if you hit any issues!
