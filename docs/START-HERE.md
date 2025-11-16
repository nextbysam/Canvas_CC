# Compositor Learning Path: The Practical Roadmap

## The Best Approach: Two-Track Learning

**Track 1: Pure DRM (Fundamentals) → Track 2: wlroots (Real Compositor)**

This gives you deep understanding + practical results.

---

## Track 1: DRM Fundamentals (Week 1-2)

### Goal: Control the screen directly, no abstractions

**Step 1: Test DRM access** (15 mins)
```bash
# On your Pi
sudo apt install libdrm-dev

# Check you can access GPU
ls -la /dev/dri/card0
sudo usermod -a -G video $USER
# Logout & login
```

**Step 2: Display a colored rectangle** (1 day)

Use this proven example: https://github.com/dvdhrm/docs/blob/master/drm-howto/modeset.c

What it teaches:
- Open `/dev/dri/card0`
- Find connected display
- Create framebuffer in memory
- Set display mode
- Draw pixels
- Display them

**Key insight:** Everything else is built on top of this!

**Step 3: Modify it** (2-3 days)
- Change colors
- Draw patterns
- Implement double buffering
- Add VSync

You now understand the foundation. Move to Track 2.

---

## Track 2: wlroots Compositor (Week 3-8)

### Goal: Build a working compositor using wlroots

**Why wlroots?**
- Handles DRM/input/protocols for you
- ~300 lines for basic compositor vs 3000+ from scratch
- Used by Sway, Labwc (your Pi uses this!)
- Best path to learning

**Step 1: Install & study tinywl** (2 days)
```bash
cd ~/
git clone https://gitlab.freedesktop.org/wlroots/wlroots.git
cd wlroots

# Build
meson build
ninja -C build

# Study the minimal example
cd tinywl
cat tinywl.c  # Read every line!
```

**Step 2: Run tinywl** (1 day)
```bash
# From VT (Ctrl+Alt+F2)
cd ~/wlroots/build/tinywl
./tinywl

# From another terminal:
WAYLAND_DISPLAY=wayland-1 weston-terminal
```

If you see a terminal window, you've run a compositor!

**Step 3: Modify tinywl** (1-2 weeks)

Start simple, add features:

1. **Change background color**
   ```c
   float color[4] = {0.3, 0.3, 0.3, 1.0};  // Change these!
   wlr_renderer_clear(renderer, color);
   ```

2. **Add keyboard shortcuts**
   ```c
   // In keyboard_handle_key():
   if (sym == XKB_KEY_q) {
       wl_display_terminate(server->wl_display);
   }
   ```

3. **Implement window tiling**
   - Automatic window positioning
   - Split screen layouts

4. **Add window decorations**
   - Title bars
   - Close buttons

**Step 4: Build your own compositor** (2-4 weeks)

Now fork tinywl and customize:
- Different tiling algorithm
- Custom status bar
- Special effects
- Your own ideas!

---

## The Critical Skills (In Order)

### 1. C Programming (if rusty)
- Pointers & structs
- Function pointers (callbacks)
- Manual memory management
- Reading others' C code

### 2. DRM/KMS API ⭐ MOST IMPORTANT
**Resources:**
- dvdhrm's modeset.c (start here!)
- https://www.kernel.org/doc/html/latest/gpu/
- https://jan.newmarch.name/Wayland/DRM/

**Core concepts:**
```
CRTC → Controls display pipeline
Connector → Physical port (HDMI)
Plane → Image layer
Framebuffer → Pixel buffer in memory
```

### 3. Event Loops
Understanding `poll()` or `epoll()`:
```c
while (running) {
    poll(fds, nfds, -1);  // Wait for events
    // Process DRM events
    // Process input events
    // Process Wayland client messages
}
```

### 4. Wayland Protocol (comes later)
- Don't worry about this initially
- wlroots handles it
- Learn by reading tinywl

---

## Your 30-Day Plan

### Days 1-3: DRM Basics
- [ ] Get modeset.c compiling
- [ ] Display solid color
- [ ] Change colors
- [ ] Understand every line

### Days 4-7: DRM Advanced
- [ ] Implement double buffering
- [ ] Draw patterns/gradients
- [ ] Handle multiple displays
- [ ] Add VSync

### Days 8-10: Setup wlroots
- [ ] Clone & build wlroots
- [ ] Run tinywl successfully
- [ ] Run client apps in it
- [ ] Read tinywl.c completely

### Days 11-20: Modify tinywl
- [ ] Change colors/rendering
- [ ] Add keyboard shortcuts
- [ ] Implement window features
- [ ] Study wlroots examples

### Days 21-30: Your Compositor
- [ ] Fork tinywl
- [ ] Add one major feature
- [ ] Handle edge cases
- [ ] Polish & test

---

## Fastest Path to Results

```
Day 1: modeset.c → red screen
       ↓
Day 3: Understand DRM basics
       ↓
Day 8: tinywl running
       ↓
Day 15: Modified tinywl with custom features
       ↓
Day 30: Your own compositor working!
```

---

## Code Examples to Study (In Order)

1. **modeset.c** - Pure DRM
   https://github.com/dvdhrm/docs/blob/master/drm-howto/modeset.c
   *Read this first!*

2. **tinywl.c** - Minimal wlroots compositor
   https://gitlab.freedesktop.org/wlroots/wlroots/-/blob/master/tinywl/tinywl.c
   *Your reference implementation*

3. **Wayland from scratch** - No libraries!
   https://gaultier.github.io/blog/wayland_from_scratch.html
   *Advanced: Shows raw Wayland protocol*

4. **Labwc source** - Production compositor
   https://github.com/labwc/labwc
   *Study after you understand tinywl*

---

## Common Mistakes to Avoid

❌ Starting with Wayland protocol details
✅ Start with DRM, understand framebuffers

❌ Building everything from scratch first
✅ Use wlroots, understand one layer at a time

❌ Reading docs without coding
✅ Code first, read docs when stuck

❌ Trying to understand everything at once
✅ Focus on one component at a time

---

## The Minimal Knowledge Stack

```
Layer 3: Wayland Protocol ──┐
                            │
Layer 2: wlroots ───────────┤ Start here (tinywl)
                            │
Layer 1: DRM/KMS ───────────┘ Understand this first

Layer 0: Linux/C ───────────── Prerequisites
```

**Study bottom-up, build top-down!**

---

## Resources (Only What You Need)

**DRM:**
- modeset.c walkthrough: https://github.com/dvdhrm/docs
- Kernel docs: https://www.kernel.org/doc/html/latest/gpu/drm-kms.html

**wlroots:**
- tinywl: wlroots/tinywl/tinywl.c
- Drew DeVault's series: https://drewdevault.com/2018/02/17/Writing-a-Wayland-compositor-1.html

**General:**
- Wayland book: https://wayland-book.com/ (reference, not tutorial)
- LWN articles: Search "Linux graphics stack"

---

## Quick Start Commands

```bash
# Get started RIGHT NOW:
mkdir ~/compositor && cd ~/compositor

# 1. Get DRM example
wget https://raw.githubusercontent.com/dvdhrm/docs/master/drm-howto/modeset.c

# 2. Compile it
gcc modeset.c -o modeset -ldrm

# 3. Run it (from VT, Ctrl+Alt+F2)
sudo ./modeset

# See colored screen? You control the GPU!
# Press Enter to exit

# 4. Next: Clone wlroots
cd ~/
git clone https://gitlab.freedesktop.org/wlroots/wlroots.git
cd wlroots && meson build && ninja -C build

# You're on your way!
```

---

## The Truth About Time Investment

**Minimal working knowledge:** 1 week
- Run modeset.c
- Run tinywl
- Basic understanding

**Comfortable understanding:** 1 month
- Modify tinywl confidently
- Add features
- Debug issues

**Deep expertise:** 3-6 months
- Build from scratch
- Understand all protocols
- Production-quality code

**Start small. Build up. Code every day.**

---

## TL;DR - Just Tell Me What To Do!

1. **Week 1:** Get modeset.c working, display colored screen
2. **Week 2:** Build & run tinywl, launch apps in it
3. **Week 3-4:** Modify tinywl, add features
4. **Beyond:** Build your vision

**First step:** Download modeset.c and make it display something. Do this TODAY.

Everything else follows naturally from understanding that one example.
