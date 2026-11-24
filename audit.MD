# Security Audit

**Repository:** AlainPlusPlus/esphome-for-watchy  
**Scope:** Review of the checked-in configuration and automation files at commit `c72df4ed1a7100902731a8c9b1e53366059fdf55`.  
**Status:** Static review; no runtime or hardware testing was performed.

## Summary

The repository is an ESP32 firmware/watch project for personal use. The main security concerns are plaintext HTTP weather requests, configuration defaults that could be mistaken for real credentials, and ingestion of large external JSON without sufficiently defensive validation. The ESPHome API and OTA components are protected by keys/passwords supplied in local secrets, but the example and documentation should discourage weak or placeholder values.

## Findings

### 1. Weather data is fetched over plaintext HTTP

**Severity:** Medium  
**Affected files:** `watchy-v3.yaml`, `watchy-v2.yaml`

The direct weather request uses:

```text
http://api.openweathermap.org/data/2.5/forecast
```

and `http_request.verify_ssl` is set to `false`.

**Risk:** Network observers can read the request and API key, and an on-path attacker can modify the returned forecast. Weather responses are not strongly integrity-protected. Even though weather data is not usually high sensitivity, this can expose the API key and location city ID.

**Recommendation:** Prefer HTTPS for OpenWeatherMap. Set `verify_ssl: true` and, on ESP devices, ensure the trust store/current root certificates are available. Use a restricted API key and monitor usage. If HTTP is retained, document that it is only for compatibility and is unauthenticated/insecure.

### 2. Example OTA password is weak and predictable

**Severity:** High  
**Affected file:** `config-example.yaml`

The example contains:

```yaml
OTA_PASSWORD: "asdf1234"
```

**Risk:** Anyone obtaining the example or devices deployed without changing it could attempt OTA access to overwrite firmware. OTA passwords should be generated per-device or at least unique before flashing.

**Recommendation:** Do not ship a usable-looking default OTA password. Use an explicit placeholder such as `REPLACE_WITH_A_RANDOM_PASSWORD`, and document that `config.yaml` must contain a strong unique value. If supported, prefer OTA mechanisms with modern authentication and restrict network exposure.

### 3. Placeholder Home Assistant API key could become a weak credential

**Severity:** Medium  
**Affected file:** `config-example.yaml`

The example defines:

```yaml
HOME_ASSISTANT_API_KEY: "your-random-key-here"
```

**Risk:** The API key is meaningful only if the user's real `config.yaml` replaces the placeholder. If left unchanged, devices may use a publicly known value and fail to authenticate or be vulnerable to unintended control where the same placeholder was reused.

**Recommendation:** Use a conspicuous placeholder that cannot be confused with a valid key, and add startup validation or installation guidance requiring a cryptographically random key. Do not commit real `config.yaml`, secrets, or generated credentials.

### 4. Agenda and weather JSON are accepted through Home Assistant services with limited validation

**Severity:** Medium  
**Affected files:** `watchy-v3.yaml`, `watchy-v2.yaml`, `automations/watchy-calendar-agenda.yaml`, `automations/watchy-weather.yaml`

The firmware exposes ESPHome services that accept JSON strings:

- `set_agenda`
- `set_weather`

The agenda parser reserves 1024 bytes and then directly converts array values. The calendar automation builds JSON from Home Assistant calendar data, while external weather/automation data is passed to the device.

**Risk:** Malformed, unexpected, or oversized service payloads can cause parse failures or memory pressure on a constrained ESP32. A second impact is that untrusted calendar titles may contain very long strings; although this code prints them to the display rather than executing them, unbounded data can consume flash/RAM or cause layout issues.

**Recommendation:** Validate payload shape, element types, numeric ranges, and maximum string lengths before storing or rendering. Reject invalid JSON and valid JSON with unexpected content. Keep a strict total agenda/response size limit and treat parse failures as a safe display state.

### 5. Calendar automation can construct invalid JSON for some values

**Severity:** Low to Medium  
**Affected file:** `automations/watchy-calendar-agenda.yaml`

The automation uses a Jinja-generated array and passes `(i.summary | trim)` through `to_json`. It is generally safer than manual escaping, but the template's commas and array data rely on Jinja rendering. If Home Assistant changes return types or event data is malformed, an incomplete or invalid array could be sent.

**Risk:** Agenda parsing degrades to an error state instead of safely displaying known-good entries. It can also create repeated `set_agenda` failures or consume parsing memory.

**Recommendation:** First build a list with Jinja and serialize it once, rather than mixing handwritten array syntax and `to_json`. Add length limits and explicit type/range checks before sending. The existing `data_template` and `to_json` usage should be tested with apostrophes, quotes, Unicode, newline characters, and empty summaries.

### 6. Direct HTTP weather endpoint is not limited to an allowlisted host or response status handling

**Severity:** Medium  
**Affected file:** `watchy-v3.yaml`

The weather script logs the response but contains a TODO for checking the HTTP status code. DNS or TLS (if enabled) aside, HTTP status is not enforced in the parser.

**Risk:** Error pages or unexpected OpenWeatherMap responses could be parsed or stored incorrectly. Without strict status and schema checking, operational failures may appear as weather corruption or consume device memory.

**Recommendation:** Check the HTTP status before parsing, require a 200 response, validate `list`, `dt`, `weather[0].id`, `main.feels_like`, and `sys.pod`, and reject unknown/missing fields safely.

### 7. Improv Serial enables unauthenticated Wi-Fi provisioning

**Severity:** Medium  
**Affected file:** `watchy-v3.yaml`

Improv Serial is enabled. It permits a connected USB/serial client to update Wi-Fi credentials. This is intended for provisioning, but a locally connected untrusted host can change network credentials while the device is in bootloader/serial mode.

**Risk:** Unauthorized physical or USB access can reconfigure network access or redirect the device to attacker-controlled infrastructure.

**Recommendation:** Disable Improv Serial after provisioning or make provisioning time-limited. Restrict physical access and document that serial access is a trusted-channel operation. Consider adding an explicit re-enable mechanism.

### 8. Wi-Fi SSID and credentials can persist outside source control

**Severity:** Low to Medium  
**Affected files:** `.gitignore`, `config-example.yaml`

The repository correctly ignores `config.yaml`, `secrets.yaml`, and `.env*`. However, no secrets template is enforced, and users might copy real values into `config-example.yaml`.

**Risk:** Accidental commits could expose Wi-Fi passwords, OTA passwords, API keys, or location data.

**Recommendation:** Keep secrets out of all tracked YAML. Add CI secret-pattern scanning and a `secrets.yaml` example containing placeholders only. Consider publishing a checksum or warning for users upgrading with old credentials.

### 9. External ESPHome component uses a mutable `dev` branch

**Severity:** Medium  
**Affected file:** `watchy-v3.yaml`

The external component is fetched from a Git URL with:

```yaml
ref: dev
```

**Risk:** Builds are not reproducible and can silently change. A compromised upstream branch or unexpected upstream update can introduce malicious code into firmware built later.

**Recommendation:** Pin the external component to an immutable commit SHA or reviewed release/tag. Document and review upstream diffs before upgrades.

### 10. Potential integer and bounds issues in display/weather logic

**Severity:** Low  
**Affected files:** `watchy-v3.yaml`, `watchy-v2.yaml`, `util.h`, `icons.h`

Several code paths index arrays using untrusted or derived values, for example weather codes/timestamps and agenda elements. The repository assumes valid array lengths and numeric values. It also calculates timer and display values using 32-bit integers.

**Risk:** Malformed API responses or service payloads can trigger out-of-bounds array access, negative timer calculations, or display overflow. On ESP32, memory corruption or crashes can occur.

**Recommendation:** Before every array access, validate indexes and lengths. Normalize/clamp numeric values and check for integer overflow. Replace direct map/array indexing with checked helpers. Add bounds assertions or safe default values (for example, the `mdiAlert` icon for unknown weather codes).

## Positive Security Observations

- `OTA_PASSWORD` is configured and the ESPHome API uses an encryption key from local configuration.
- Secrets and local configuration files are ignored by Git.
- Logger output is disabled.
- Agenda JSON is capped by a fixed buffer in the display/parser code.
- HTTP weather responses have a 3 KB response limit.
- The API exposes only explicit services (`set_weather` and `set_agenda`) rather than a broad shell or filesystem interface.

## Prioritization

1. Replace predictable example OTA credentials and prevent placeholder credentials from being used.
2. Move the weather request to verified HTTPS and validate response status/schema.
3. Pin the external ESPHome component to an immutable commit.
4. Tighten agenda/weather input validation and bounds checking.
5. Decide whether Improv Serial remains enabled after provisioning.
6. Add secret scanning and generated-configuration checks to CI.

## Limitations

This review was based on repository source available at the referenced commit. It does not establish exploitability at runtime, verify ESPHome framework behavior, or confirm the security behavior of the external `dev` branch. Hardware and network attack testing would be required for an operational security assessment.
