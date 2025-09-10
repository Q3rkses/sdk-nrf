# Baseline Channel Filter Algorithm Hardware Sample

## Overview

This is a hardware sample implementation of the autonomous channel map update algorithm for Bluetooth Low Energy (BLE) connections using QoS events. The algorithm automatically evaluates channel performance and generates optimized channel maps to improve connection reliability. This sample is intended to be run on Nordic Semiconductor hardware.

## Key Features

- **Autonomous Operation**: Automatically collects QoS data and updates channel maps.
- **Configurable Parameters**: Adjustable weights and thresholds for different environments.
- **Rating System**: Each channel gets a performance rating based on CRC errors and RX timeouts.
- **Minimum Channel Guarantee**: Ensures a minimum number of active channels for connectivity.

## Algorithm Description

### Rating Calculation

The algorithm evaluates each channel after collecting a configurable number of samples using the following formula:

```
rating = (1.0f - w_3) * prev_rating + w_3 * (1.0f - (w_1 * crc_error_rate + w_2 * rx_timeout_rate));
```

Where:
- `w_1`: Weight for CRC errors
- `w_2`: Weight for RX timeouts
- `w_3`: Weight for old rating
- `crc_error_rate`: `crc_errors / packets_sent`
- `rx_timeout_rate`: `rx_timeouts / packets_sent`
- `prev_rating`: The rating from the previous evaluation.

### Channel Map Generation

Channels are included in the channel map if:
1. Their rating is above a configurable threshold (default: 0.3).
2. At least 2 channels are always kept active (as a safety mechanism).

---

## Building, Flashing, and Running

This sample targets Nordic Semiconductor devices and has been tested with **nRF Connect SDK v2.6** (Zephyr v3.5).

### Build and Flash

Compile and flash for a physical DK (the example uses the nRF54l15DK):

```bash
# Build the sample
west build -b nrf54l15dk/nrf54l15/cpuapp/ autonomous_channel_map_filter_hardware_sample
or
west build -b nrf54l15dk/nrf54l15/cpuapp/ns autonomous_channel_map_filter_hardware_sample

# Flash the sample
west flash
```

### Running the Sample

1.  **Flash** two development kits with the firmware.
2.  Open a serial terminal to each board (e.g., using `nrfjprog -r -c 1000000 -t NRF52`).
3.  On one terminal, start advertising. On the other, start scanning and connect.
4.  While connected, the central will gather statistics.
5.  After the evaluation interval is reached, the algorithm will run and print the results.

### Example Output

```
*** Booting Zephyr OS build v3.5.0-4634-g22246087599c ***
Starting Channel Map Filter Demo
Bluetooth initialized
Scanning successfully started
[DEVICE]: C6:26:55:41:35:54 (random), connectable, tx_power 0, rssi -36
Connecting to C6:26:55:41:35:54 (random)
Connected: C6:26:55:41:35:54 (random)
QoS Initialized
--- ALGORITHM EVALUATION #1: Channel map updated. Active channels: 37 ---

--- ALGORITHM EVALUATION #2: Channel map updated. Active channels: 36 ---

...
```

---
## API Reference

### Structures

```c
struct chmap_filter_params {
    float w_0;                        // Weight for temporary rating
    float w_1;                        // Weight for CRC errors
    float w_2;                        // Weight for RX timeouts
    float w_3;                        // Weight for old rating
    float rating_threshold;           // Threshold for disabling channels
    uint16_t min_packets_per_channel; // Minimum packets needed for rating
};

struct chmap_qos_sample {
    uint8_t channel_index;    // Channel index (0-36)
    uint16_t packets_sent;    // Number of packets sent
    uint16_t crc_ok_count;    // Number of packets with CRC OK
    uint16_t crc_error_count; // Number of packets with CRC errors
    uint16_t rx_timeout_count; // Number of RX timeouts
};
```

### Functions

```c
// Initialize the algorithm
void baseline_ch_filter_algo_init(struct chmap_instance *chmap_instance);

// Check if algorithm is ready for evaluation and perform it
int baseline_ch_filter_algo_evaluate(struct chmap_instance *chmap_instance);

// Get the suggested channel map
uint8_t *baseline_ch_filter_algo_get_channel_map(struct chmap_instance *chmap_instance);

// Set algorithm parameters
void baseline_ch_filter_algo_set_parameters(struct chmap_instance *chmap_instance,
                                           const struct chmap_filter_params *params);
// Helper function to create a QoS sample
static inline void baseline_ch_filter_algo_create_sample(struct chmap_qos_sample *sample,
                                                             uint8_t channel_index,
                                                             uint16_t packets_sent,
                                                             uint16_t crc_ok_count,
                                                             uint16_t crc_error_count,
                                                             uint16_t rx_timeout_count)
```

### Usage Example

```c
#include "final_algo.h"

// Global instance
static struct chmap_instance g_chmap_instance;

// Initialize during startup
void bluetooth_init(void)
{
    baseline_ch_filter_algo_init(&g_chmap_instance);
}

// In your QoS event handler
void on_qos_event(uint8_t channel, uint16_t crc_ok, uint16_t crc_error, uint16_t rx_timeout)
{
    struct chmap_qos_sample sample;
    
    // Create sample (assuming 1 packet per event)
    baseline_ch_filter_algo_create_sample(&sample, channel, 1, crc_ok, crc_error, rx_timeout);

    // This part is handled inside the evaluate function in this version of the algorithm
    // baseline_ch_filter_algo_add_sample(&g_chmap_instance, &sample);
    
    // Check if evaluation is ready
    int result = baseline_ch_filter_algo_evaluate(&g_chmap_instance);
    if (result == 1) {
        // Apply new channel map
        const uint8_t *new_map = baseline_ch_filter_algo_get_channel_map(&g_chmap_instance);
        bt_le_set_chan_map(new_map);
    }
}
```

---

## Configuration

### Default Parameters
- **Evaluation Interval**: 2000 packets (total)
- **Minimum Packets per Channel**: 10
- **Rating Threshold**: 0.7
- **Weight `w_1` (CRC errors)**: 0.2
- **Weight `w_2` (RX timeouts)**: 0.8
- **Weight `w_3` (Old rating)**: 0.85
- **Minimum Active Channels**: 5
- **Channel Cooldown Time**: 2 evaluation cycles

### Tuning Guidelines
- **Increase `w_1`** for environments with high CRC error rates
- **Increase `w_2`** for environments with high RX timeout rates
- **Increase `rating_threshold`** for more aggressive channel filtering
- **Decrease `w_3`** to respond faster to channel changes
- **Increase `min_packets_per_channel`** for more reliable statistics

## Integration Notes

### For Zephyr/NCS
```c
// In your QoS event handler
static bool on_vs_evt(struct net_buf_simple *buf)
{
    // ... existing code ...
    
    struct chmap_qos_sample sample;
    baseline_ch_filter_algo_create_sample(&sample, 
                                         evt->channel_index,
                                         1,  // One packet per event
                                         evt->crc_error_count > 0 ? 0 : 1,
                                         evt->crc_error_count,
                                         evt->rx_timeout);
    
    baseline_ch_filter_algo_add_sample(&g_chmap_instance, &sample);
    
    if (baseline_ch_filter_algo_evaluate(&g_chmap_instance) == 1) {
        const uint8_t *new_map = baseline_ch_filter_algo_get_channel_map(&g_chmap_instance);
        bt_le_set_chan_map(new_map);
    }
    
    return true;
}
```

### Memory Usage
- **Instance Size**: ~1.5 KB per instance
- **Stack Usage**: Minimal (~100 bytes for function calls)
- **Heap Usage**: None (all static allocation)

## Performance Characteristics

- **Evaluation Time**: ~1ms for 37 channels on modern processors
- **Memory Access**: Mostly sequential, cache-friendly
- **Computation**: Lightweight floating-point operations
- **Latency**: Evaluation triggered every 2000 samples

## Limitations

1. **Fixed Sample Count**: Currently evaluates every 2000 samples
2. **Simple Algorithm**: Basic weighted average, no advanced ML
3. **No Frequency Analysis**: Doesn't consider frequency domain effects
4. **Static Thresholds**: Fixed thresholds may not adapt to all environments

## Future Enhancements

- Adaptive sample count based on channel stability
- Machine learning for pattern recognition
- Frequency domain analysis for interference detection
- Dynamic threshold adjustment
- Multi-connection channel map coordination

## Contributing

This is a baseline implementation intended for:
1. **Research**: Understanding channel behavior
2. **Benchmarking**: Comparing against advanced algorithms
3. **Prototyping**: Quick integration and testing
4. **Education**: Learning channel map algorithms

Feel free to extend and improve the algorithm for your specific use case.

## License

This implementation is provided as-is for research and development purposes. 
