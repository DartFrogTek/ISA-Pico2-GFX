# ISA-Pico2-GFX
Future project: An ISA card that can output DVI via HDMI with HSTX on Pico2 (RP2350)

---

### Essential ISA Bus Signals
- Address Lines (minimum needed)
  - A0-A9: For I/O port decoding (gives 1024 possible I/O addresses)
  - A10-A15: Optional, if wanted more I/O address space
  - A0-A19: Memory-mapped framebuffer access (full 1MB ISA memory space)

- Data Lines
  - D0-D7: 8-bit data bus (minimum for basic operation)
  - D8-D15: 16-bit ISA support (recommended)

- Control Signals (critical)
  - IOR#: I/O Read strobe - tells when CPU is reading from I/O ports
  - IOW#: I/O Write strobe - tells when CPU is writing to I/O ports
  - MEMR#: Memory Read - if implementing memory-mapped access
  - MEMW#: Memory Write - if implementing memory-mapped access

- Bus Control
  - AEN: Address Enable - goes high during DMA cycles (typically ignore bus when this is active)
  - CLK: ISA bus clock (~8.33MHz) - for timing reference
  - RESET: System reset signal

- Optional but Useful
  - IOCS16#: Pull low to tell system to support 16-bit transfers
  - CHRDY: Channel Ready - pull low to insert wait states if RP2350 needs more time
  - IRQ3-IRQ7: If wanted interrupt capability (for VSync interrupts, etc.)

### Minimal Implementation
For a basic proof-of-concept, start with just:
- Address: A0-A9 (I/O port decoding)
- Data: D0-D7 (8-bit transfers)
- Control: IOR#, IOW#, AEN, CLK, RESET
This gives basic register access for configuring video modes and writing palette data.

### RP2350 GPIO Allocation
Need about 20-25 GPIO pins for ISA interface:
- 10 pins for A0-A9
- 8 pins for D0-D7
- 5 pins for control signals
- Plus DVI output pins (4 differential pairs + clock)

The Pico Plus 2 has enough GPIOs to handle this comfortably.

---

## I/O Port Mapping Strategy

**ISA I/O Address Space Layout**
ISA uses a 16-bit I/O address space (0x0000-0xFFFF), but many ranges are reserved:
- 0x000-0x0FF: System/motherboard
- 0x100-0x1FF: Available for expansion cards
- 0x200-0x2FF: Game ports, some sound cards
- 0x300-0x3FF: **Common graphics card range** (VGA uses 0x3C0-0x3DF)

**Recommended Base Addresses for Card**
```
Primary Option:   0x300-0x30F (16 registers)
Secondary Option: 0x310-0x31F (avoid conflicts with VGA)
Tertiary Option:  0x240-0x24F (less common range)
```

## Address Decoding Logic

**Hardware Approach (Simple but Limited)**
Use discrete logic to decode a fixed base address:
```
Base = 0x300 (example)
A9 A8 A7 A6 A5 A4 A3 A2 A1 A0
 1  1  0  0  0  0  0  0  X  X  = 0x300-0x303

Decode Logic:
CARD_SELECT = A9 & A8 & !A7 & !A6 & !A5 & !A4 & !A3 & !A2 & !AEN
```

**Software Approach (Flexible - Recommended)**
Let the RP2350 do full address decoding:
```c
// In ISA bus handler interrupt
uint16_t io_address = read_address_bus();
uint8_t data = read_data_bus();

if ((io_address & 0xFF0) == 0x300) {  // Base address 0x300-0x30F
    uint8_t reg = io_address & 0x0F;   // Register offset 0-15
    handle_register_access(reg, data);
}
```

## Register Map Layout

**Proposed Register Mapping (Base + Offset)**
```
0x300 (Base+0):  Control Register
  Bit 0: Enable output
  Bit 1: Reset video timing
  Bit 2-4: Video mode select (0-7)
  Bit 5: Interrupt enable
  Bit 6-7: Reserved

0x301 (Base+1):  Status Register (Read Only)
  Bit 0: VSync active
  Bit 1: HSync active  
  Bit 2: Display enabled
  Bit 3: DVI link active
  Bit 4-7: Reserved

0x302 (Base+2):  Palette Address
0x303 (Base+3):  Palette Data (RGB)

0x304 (Base+4):  Framebuffer Address Low
0x305 (Base+5):  Framebuffer Address High
0x306 (Base+6):  Framebuffer Address Extended

0x307 (Base+7):  H-Resolution Low
0x308 (Base+8):  H-Resolution High
0x309 (Base+9):  V-Resolution Low
0x30A (Base+10): V-Resolution High

0x30B (Base+11): Pixel Format
  0 = 1bpp, 1 = 4bpp, 2 = 8bpp, 3 = 16bpp, 4 = 24bpp

0x30C-0x30F:     Reserved for expansion
```

## RP2350 Implementation

**GPIO Setup**
```c
// Address bus pins
#define ADDR_BASE_PIN 0   // A0-A9 on GPIO 0-9
#define DATA_BASE_PIN 10  // D0-D7 on GPIO 10-17

// Control pins  
#define IOR_PIN    18
#define IOW_PIN    19
#define AEN_PIN    20
#define CLK_PIN    21
#define RESET_PIN  22
```

**Interrupt-Driven Bus Handler**
```c
void setup_isa_interface() {
    // Set address/data pins as inputs with pull-ups
    for(int i = 0; i < 10; i++) {
        gpio_init(ADDR_BASE_PIN + i);
        gpio_set_dir(ADDR_BASE_PIN + i, GPIO_IN);
        gpio_pull_up(ADDR_BASE_PIN + i);
    }
    
    // Setup interrupt on IOR# and IOW# falling edges
    gpio_set_irq_enabled_with_callback(IOR_PIN, GPIO_IRQ_EDGE_FALL, 
                                       true, &isa_bus_callback);
    gpio_set_irq_enabled_with_callback(IOW_PIN, GPIO_IRQ_EDGE_FALL, 
                                       true, &isa_bus_callback);
}

void isa_bus_callback(uint gpio, uint32_t events) {
    if (gpio_get(AEN_PIN)) return; // Ignore during DMA
    
    uint16_t address = read_address_bus();
    
    if ((address & 0xFF0) == ISA_BASE_ADDR) {
        if (gpio == IOR_PIN) {
            handle_io_read(address & 0x0F);
        } else if (gpio == IOW_PIN) {
            handle_io_write(address & 0x0F, read_data_bus());
        }
    }
}
```

**Address Decoding Options**
1. **Fixed decode**: Hardwire to one base address (simple)
2. **Jumper selectable**: Use jumpers to select from 2-4 base addresses
3. **Software configurable**: Store base address in EEPROM/flash

---

## ISA Bus Interface Refinements

**Voltage Level Considerations**
The biggest challenge will be level shifting - ISA bus operates at 5V TTL levels while the RP2350 is 3.3V. Needed:
- Bidirectional level shifters for data bus (D0-D7/D15)
- Input level shifters for address and control signals
- Consider 74LVC245 or similar for data bus
- 74LVC16244 for address/control inputs

**Timing Considerations**
ISA bus timing can be tight (~125ns access time). The previous interrupt-driven approach might be too slow. Consider:
```c
// Polling approach for better timing
void isa_bus_poll() {
    static uint32_t last_state = 0;
    uint32_t current_state = (gpio_get(IOR_PIN) << 1) | gpio_get(IOW_PIN);
    
    if ((last_state & 0x02) && !gpio_get(IOR_PIN) && !gpio_get(AEN_PIN)) {
        // IOR# falling edge - handle read immediately
        handle_io_read_fast();
    }
    last_state = current_state;
}
```

## Video Output Optimization

**HSTX DVI Implementation**
Bandwidth analysis. For practical implementation:

```c
// Optimized for 800x600@60Hz 16bpp (more achievable bandwidth)
#define VIDEO_WIDTH     800
#define VIDEO_HEIGHT    600
#define PIXEL_CLOCK_MHZ 40
#define BYTES_PER_PIXEL 2

// DMA descriptor for scanline streaming
typedef struct {
    uint32_t* psram_addr;
    uint16_t pixel_count;
    uint8_t  line_repeat;  // For resolution scaling
} scanline_desc_t;
```

**Color Depth Strategy**
Start with these modes for practical bandwidth:
- 640x480x16bpp (RGB565) - ~37MB/s
- 800x600x8bpp (palette) - ~29MB/s  
- 1024x768x8bpp (palette) - ~47MB/s

## Memory Architecture

**PSRAM Layout Optimization**
```c
#define FB_640x480x16   0x000000  // 614KB
#define FB_800x600x8    0x096000  // 480KB
#define FB_1024x768x8   0x112000  // 786KB
#define PALETTE_BASE    0x1F0000  // 256*3 bytes per palette
#define PATTERN_CACHE   0x1F8000  // For common patterns/fonts
```

**DMA Chain Architecture**
```c
// Two-channel approach
dma_channel_config cfg_pixels = dma_channel_get_default_config(dma_chan_pixels);
channel_config_set_transfer_data_size(&cfg_pixels, DMA_SIZE_16);
channel_config_set_read_increment(&cfg_pixels, true);
channel_config_set_write_increment(&cfg_pixels, false);
channel_config_set_dreq(&cfg_pixels, DREQ_HSTX);

// Chain to handle horizontal blanking automatically
dma_channel_config cfg_blank = dma_channel_get_default_config(dma_chan_blank);
channel_config_set_chain_to(&cfg_pixels, dma_chan_blank);
```

## Register Interface Enhancements

**Extended Register Map**
```c
// Add these to register map:
#define REG_DMA_CTRL     0x30C  // DMA enable/status
#define REG_LINE_OFFSET  0x30D  // Bytes per scanline (for scrolling)
#define REG_START_ADDR   0x30E  // Display start address (paging)
#define REG_CURSOR_CTRL  0x30F  // Hardware cursor control
```

**Bus Mastering Consideration**
While ISA doesn't have true bus mastering, could implement a "fast blit" mode:
```c
// Host writes source/dest/size, then triggers DMA copy
typedef struct {
    uint32_t src_addr;
    uint32_t dst_addr; 
    uint16_t width, height;
    uint8_t  operation;  // copy, fill, XOR, etc.
} blit_cmd_t;
```

### Implementation Suggestions

**Development Phases**
1. **Phase 1**: Basic ISA interface + 640x480x8bpp
2. **Phase 2**: Add DVI output with fixed test pattern
3. **Phase 3**: Integrate PSRAM framebuffer
4. **Phase 4**: Higher resolutions and color depths
5. **Phase 5**: Hardware acceleration features

**PCB Design Considerations**
- Keep ISA signals short and well-terminated
- Separate analog (DVI) and digital power planes
- Consider PSRAM placement near RP2350 for signal integrity
- Add test points for debugging ISA timing

**Software Driver Strategy**
DOS/Windows drivers. Consider:
- DOS: Direct I/O port access, easy to debug
- Windows 3.1: VGA-compatible mode for basic operation
- Modern systems: Write a simple framebuffer driver

## Bandwidth Workarounds

**Compression Techniques**
```c
// Run-length encoding for common patterns
typedef struct {
    uint16_t color;
    uint16_t count;
} rle_pair_t;

// Text mode optimization - store characters + attributes
typedef struct {
    uint8_t character;
    uint8_t attribute;
} text_cell_t;
```

---

### PSRAM + DMA for Video Framebuffer
- Advantages of External PSRAM
  - 8MB gives plenty of room for multiple framebuffers (1920x1080x24bpp = ~6MB)
  - Can store multiple video modes simultaneously
  - Leaves internal SRAM free for code and DMA descriptors
  - Allows for double/triple buffering to eliminate tearing

### DMA Implementation
- The RP2350's DMA controllers can definitely access PSRAM:
  - Use DMA to continuously stream pixel data from PSRAM to HSTX
  - Set up circular DMA transfers for continuous video output
  - PSRAM is accessed via QSPI, which the DMA can handle efficiently

### Bandwidth Considerations
  - PSRAM typically runs at 80-133MHz quad SPI
  - For 1080p@60Hz (148.5MHz pixel clock), need ~445MB/s for 24bpp
  - RP2350 PSRAM interface can typically achieve 80-100MB/s:
    - Use lower color depths (16bpp or 8bpp with palette)
    - Implement compression or run-length encoding
    - Use lower resolutions initially

### Practical Implementation
- PSRAM Layout Example:
  - 0x000000-0x4FFFF: 640x480x16bpp framebuffer
  - 0x050000-0x1FFFFF: 1024x768x16bpp framebuffer  
  - 0x200000-0x7FFFFF: Additional buffers/palettes
  
### DMA Chain Setup
- Configure DMA to read scanlines from PSRAM
- Chain multiple DMA channels: one for pixel data, another for sync signals
- Use DMA interrupts to handle vertical blanking periods

### Memory Management
- ISA bus writes go directly to PSRAM via DMA
- Use memory mapping to let the ISA host write to different video modes
- Implement page flipping by changing DMA source addresses

# Board
- Pimoroni Pico Plus 2
  - RP2350 microcontroller with 16MB of flash memory, 8MB of PSRAM, USB-C, Qw/ST and debug connectors.

# Resources
https://www.cnx-software.com/2024/08/15/raspberry-pi-rp2350-hstx-high-speed-serial-transmit-interface/
