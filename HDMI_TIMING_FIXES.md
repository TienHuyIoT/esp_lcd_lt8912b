# HDMI Timing Analysis And Fix Notes

## Summary

This note records the HDMI timing issue analysis for the ESP32-P4 + LT8912B path and the fixes applied in this project.

The core issue was not a generic LT8912B driver failure. The main problem was that several HDMI timing definitions used pixel clocks that do not map cleanly onto the ESP32-P4 clocking constraints.

## Root Cause

For this hardware path, the effective constraint is:

- DPI clock is derived from `PLL_F240M` divided by an integer.
- For 2 DSI lanes and 24bpp, lane bit rate must stay aligned with the pixel clock ratio.
- Practical stable pixel clocks on ESP32-P4 are multiples of 5 MHz, especially `20`, `30`, `40`, `60`, `80` MHz.

Several existing LT8912B presets in the BSP used values such as:

- `56 MHz`
- `64 MHz`
- `70 MHz`

Those values do not fit the expected clean divider behavior for the ESP32-P4 HDMI path. In practice this can lead to:

- unstable or missing HDMI output
- skewed or corrupted image timing
- modes that appear valid on paper but do not lock reliably on a monitor

This matches the external report and the working results from the reference implementation in:

- `https://github.com/troyhacks/WLED/blob/Olimex_HDMI_Output/wled00/wled_hdmi.cpp`

## Important Finding

The issue is not only the nominal resolution. The porch and sync values also matter because the LT8912B and ESP32-P4 need timings that land on workable clock ratios.

The reference implementation showed that some non-standard but monitor-valid timings work much better than the original defaults.

Working modes highlighted by that report included:

- `720x576@50`
- `800x600@60`
- `1024x576@57`
- `1280x720@60`
- `1280x720@50`
- `1280x800@50`

This project only exposed a smaller set of HDMI resolutions in the BSP, so the applied patch focused on correcting the existing supported modes rather than expanding the public BSP API.

## Files Changed

- `components/esp-bsp/components/lcd/esp_lcd_lt8912b/include/esp_lcd_lt8912b.h`
- `components/esp-bsp/bsp/esp32_p4_function_ev_board/esp32_p4_function_ev_board.c`

## Fixes Applied

### 1. Fixed LT8912B timing presets

The following timing presets were corrected in `esp_lcd_lt8912b.h`.

#### 800x600

Changes:

- `hfp`: `48 -> 40`
- aspect ratio: `16:9 -> 4:3`

Reason:

- The original front porch did not match the declared total timing.
- Correct total is `800 + 40 + 128 + 88 = 1056`.

#### 1024x768

Changes:

- pixel clock: `56 -> 60 MHz`
- `vs`: `4 -> 5`
- `vbp`: `15 -> 69`
- `vtotal`: `790 -> 845`
- aspect ratio: `16:9 -> 4:3`

Reason:

- `56 MHz` is not a stable target for this ESP32-P4 HDMI path.
- The replacement timing matches a valid `60 MHz` clock domain and stable totals.

#### 1280x720

Changes:

- pixel clock: `64 -> 60 MHz`
- `hfp`: `48 -> 10`
- `hbp`: `80 -> 28`
- `htotal`: `1440 -> 1350`
- `vic`: `0 -> 4`

Reason:

- `64 MHz` was one of the main invalid presets.
- The updated mode follows the known working `1280x720@60` timing from the reference implementation.
- `VIC 4` matches standard 720p60 signaling.

#### 1280x800

Changes:

- pixel clock: `70 -> 60 MHz`
- `vs`: `6 -> 5`
- `vbp`: `14 -> 25`
- `vtotal`: `823 -> 833`
- aspect ratio: `16:9 -> no data`

Reason:

- `70 MHz` is not a clean supported value for this path.
- `1280x800` is `16:10`, so the previous aspect ratio metadata was also wrong.
- With the corrected totals, the actual refresh is roughly `50 Hz`.

#### 1920x1080@30

Changes:

- pixel clock: `70 -> 80 MHz`
- `vbp`: `8 -> 194`
- `vtotal`: `1096 -> 1282`

Reason:

- `70 MHz` was invalid here too.
- `80 MHz` is the closest workable clean clock.

Limit:

- `1080p` is still risky on ESP32-P4 because `80 MHz` pushes DMA bandwidth hard.
- The mode now matches the clocking logic better, but it is still not guaranteed to be stable on all setups.

### 2. Fixed DSI lane bit rate selection for HDMI

In `esp32_p4_function_ev_board.c`, the DSI bus config previously reused a generic lane bit rate from the BSP default:

- `BSP_LCD_MIPI_DSI_LANE_BITRATE_MBPS = 1000`

That was too blunt for the HDMI cases.

The patch now overrides `lane_bit_rate_mbps` per HDMI mode before creating the DSI bus:

- `800x600 -> 480 Mbps`
- `1024x768 -> 720 Mbps`
- `1280x720 -> 720 Mbps`
- `1280x800 -> 720 Mbps`
- `1920x1080 -> 960 Mbps`

Reason:

- Lane rate must match the pixel clock and lane count.
- Formula used:

`lane_bit_rate_mbps = pixel_clock_mhz * 24 / lane_count`

For 2 lanes, that becomes:

- `40 MHz -> 480 Mbps`
- `60 MHz -> 720 Mbps`
- `80 MHz -> 960 Mbps`
