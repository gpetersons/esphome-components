# Changes Since `aronsky/esphome-components`

This document summarizes, at a coarse level, what had to be changed after restoring the repository to the `aronsky` baseline in order to make `ble_adv_controller` work again with ESPHome `2026.2.4`.

Baseline used:

- `aronsky/esphome-components` at commit `dbb580c` (`Adapt to changes in esphome 2025.7.0`)

Resulting fix series on top of that baseline:

- `bc7bb3e` `Fix ble_adv_controller for ESPHome 2026.2.4`
- `be311a5` `Fix boot-time encoder lifetime in ble_adv_controller`
- `b5da592` `Fix BLE advertising state handling`
- `6a88cd6` `Wait for BLE stack before raw advertising`
- `557c901` `Preserve configured BLE encoder variant`
- `d884c26` `Register BLE GAP handler during codegen`

## 1. Restore the aronsky baseline

The fork was first reset back to the aronsky version so that compatibility work started from the last known-good upstream shape instead of the intermediary fork state.

This removed unrelated fork changes from the debugging surface and made behavior comparison against the old working setup much clearer.

## 2. Update for ESPHome 2026 codegen and entity API changes

ESPHome `2026.2.4` changed several entity and codegen details enough that the component no longer compiled unchanged.

The component had to be adjusted for:

- dynamic `select` and `number` platform registration
- current entity setup and registration helpers
- newer `SelectTraits` option handling
- updated `select` callback signatures
- current preference restoration helpers
- current `LogStringLiteral`-based component source setup

This work is what made the external component compile again against ESPHome `2026.2.4`.

## 3. Fix boot-time crashes caused by invalid dynamic select state

After compilation worked again, the device still boot-looped.

The root cause was that the dynamic encoding select was populated from temporary string storage. After setup returned, the select contained dangling option pointers, which later restored as garbage. That corrupted the selected encoder and led to null dereferences when the light tried to send commands during boot state restore.

The fix was to:

- keep encoding option strings alive on the controller
- validate encoder refresh and encoder availability
- make command enqueue paths tolerate a missing encoder instead of panicking

## 4. Rework advertising flow for ESP-IDF 5.5 / ESPHome 2026 BLE behavior

Once the node booted, command sends still failed because the original advertising loop assumed older synchronous BLE GAP behavior.

On current ESPHome / ESP-IDF, raw advertising setup and start/stop transitions must be treated as asynchronous state changes. The component therefore had to be updated so that raw advertisement lifecycle followed the BLE GAP completion events instead of assuming that config/start/stop completed immediately in the same loop iteration.

## 5. Delay raw advertising until the BLE stack is actually active

During startup, commands could already be queued while the BLE stack was still coming up.

That caused repeated `ESP_ERR_INVALID_STATE` failures from `esp_ble_gap_config_adv_data_raw()` until the handler was gated to wait for the ESPHome BLE component to report that BLE was active.

This removed the startup advertising errors without dropping queued commands.

## 6. Preserve the configured encoder variant during startup

Another runtime regression was that the configured encoder variant could be unintentionally overwritten by the dynamic encoding select's fallback initialization.

The effect was that a controller configured for `lampsmart_pro - v3` could fall back to the synthetic `All` encoder and emit multiple variant packets for a single command.

The fix was to preserve an already-established encoder state and avoid re-defaulting the select when it already had a valid value.

## 7. Register BLE GAP event handling through ESPHome codegen

The final functional issue was integration with ESPHome's BLE event fanout.

The component depends on BLE GAP completion events to move raw advertising through:

1. configure raw payload
2. start advertising
3. stop advertising / rotate packet

The handler originally was not registered through ESPHome's BLE codegen registration path, so it could log queued packets but never receive the completion events required to actually progress the advertising lifecycle.

The fix was to:

- declare `BleAdvHandler` as an `esp32_ble.GAPEventHandler`
- register it during codegen with `esp32_ble.register_gap_event_handler(...)`

This was the final step that made commands affect the actual lamp again.

## 8. Validation approach

The component was validated in two stages:

- compile validation against the saved real ESPHome configuration on `2026.2.4`
- runtime validation against serial logs from the real ESP32 and comparison with saved known-good logs from the older working setup

The log comparison was important because several later issues were not compile failures, but behavioral regressions in:

- encoder selection
- BLE advertising sequencing
- BLE event registration

## 9. Net effect

Relative to the aronsky baseline, the repository now contains the minimum compatibility and runtime fixes needed so that:

- the component builds on ESPHome `2026.2.4`
- the ESP32 boots without looping
- commands do not fail immediately due to BLE state issues
- the configured encoder variant is preserved
- BLE raw advertising is actually driven through ESPHome's current BLE event system

That combination was required to get back to a working end-to-end setup.
