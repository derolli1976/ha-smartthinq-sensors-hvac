# SmartThinQ LGE Sensors - AI Agent Instructions

## Project Overview
This is a Home Assistant custom integration for LG ThinQ devices (washers, dryers, refrigerators, AC units, etc.). It's structured as a cloud-polling integration that wraps the LG ThinQ API via an embedded `wideq` library.

**Key Architecture:**
- `custom_components/smartthinq_sensors/` - Home Assistant integration layer
- `custom_components/smartthinq_sensors/wideq/` - Embedded LG API client library (async)
- Device-specific implementations in `wideq/devices/*.py` use a factory pattern

## Critical Setup Requirements

### Development Environment
```bash
# Install dependencies (legacy resolver required for HA compatibility)
pip3 install --use-deprecated=legacy-resolver -r requirements.txt

# For testing
pip3 install --use-deprecated=legacy-resolver -r requirements_test.txt

# Run Home Assistant dev instance (port 8123)
scripts/develop

# Run tests with coverage
pytest --cov-report term-missing -vv --durations=10
```

### Running Home Assistant
- **Standard mode:** VS Code task "Run Home Assistant on port 8123" or `scripts/develop`
- **With emulation:** Task "Run Home Assistant on port 8123 (with emulation)" sets `thinq2_emulation=ENABLED`
- Config lives in `config/` directory (auto-created), separate from `custom_components/`
- Set `PYTHONPATH="${PWD}/custom_components"` to allow HA to find the integration

## Code Architecture Patterns

### Device Factory System
Devices are instantiated via `wideq/factory.py::get_lge_device()` which returns a list of `Device` objects based on `DeviceType`. Some devices like `TOWER_WASHERDRYER` create multiple sub-devices (washer + dryer).

**Adding a new device type:**
1. Create class in `wideq/devices/<device>.py` extending `Device`
2. Add `DeviceType` enum to `wideq/device_info.py`
3. Register in `wideq/factory.py::get_lge_device()`
4. Implement platform entities in `custom_components/smartthinq_sensors/<platform>.py`

### Three-Layer Device Model
1. **wideq Device** (`wideq/device.py::Device`) - API abstraction, device state monitoring via `Monitor` class
2. **Helper Wrapper** (`device_helpers.py::LGEBaseDevice`) - Formatting, common sensor attributes
3. **HA Entity** (e.g., `sensor.py::ThinQSensor`) - Home Assistant entity implementation using `CoordinatorEntity`

### State Update Flow
```python
# LGEDevice in __init__.py coordinates updates
async def _async_state_update():
    # Polls device via wideq Monitor
    # Updates coordinator
    # Entities listen via coordinator callbacks
```

**Polling interval:** 30 seconds (`SCAN_INTERVAL = timedelta(seconds=30)`)

### Authentication & Client Management
- OAuth2-based auth via `LGEAuthentication` class
- Shared `ClientAsync` instance across all devices (singleton pattern via `Monitor._client_lock`)
- Handles auth refresh, credential errors, and disconnection recovery
- **Important:** Accounts using social login (Google/Facebook/Amazon) are NOT supported

## Home Assistant Integration Specifics

### Config Flow Pattern
- Multi-step flow: User credentials → OAuth → Token → Device discovery
- Supports both direct login and URL redirect methods (`CONF_USE_REDIRECT`)
- Region/language validation via ISO codes (ISO 3166-1 alpha-2, ISO 639-1)
- Entry stored in `ConfigEntry.data` with `CONF_TOKEN`, `CONF_REGION`, `CONF_LANGUAGE`, `CONF_OAUTH2_URL`, `CONF_CLIENT_ID`

### Platform Registration
Platforms are defined in `SMARTTHINQ_PLATFORMS` list in `__init__.py`. Each platform:
- Implements `async_setup_entry()`
- Listens to `LGE_DISCOVERY_NEW` signal for new devices
- Uses `@dataclass` descriptions (e.g., `ThinQSensorEntityDescription`) for entity definitions

### Services
Device-specific services registered via `platform.async_register_entity_service()`:
- `remote_start`, `wake_up`, `set_time` (washers/dryers)
- `set_sleep_time` (AC units)
- Service schemas defined using `vol.Schema` in each platform file

## Testing Conventions

### Test Structure
- `tests/conftest.py` provides fixtures including `auto_enable_custom_integrations`
- Mock `persistent_notification` to avoid integration dependencies
- Use `pytest-homeassistant-custom-component` fixtures
- Config flow tests in `tests/test_config_flow.py` mock `LGEAuthentication`

### Coverage Requirements
- Tests run with coverage reporting via `pytest-cov`
- Exclude patterns defined in `setup.cfg` under `[coverage:report]`

## Code Style & Tooling

### Formatting & Linting
```bash
# Configured in setup.cfg
- Black (line length 88)
- isort (profile=black, known_first_party=homeassistant)
- flake8 (ignores E501, W503, E203 for Black compatibility)
- Ruff (version 0.0.261)
```

### Version Compatibility
- **Minimum HA version:** 2025.1.0 (enforced in `const.py::MIN_HA_MAJ_VER/MIN_HA_MIN_VER`)
- Check via `is_valid_ha_version()` during setup
- Python 3.11+ (handles aiohttp cleanup_closed memory leak workarounds)

## Common Pitfalls & Gotchas

1. **Don't use `run_in_terminal` for Python code** - HA runs in a container with specific PYTHONPATH. Use pytest or the develop script.

2. **wideq API v2 specifics:**
   - All devices use ThinQ2 protocol (`PlatformType.THINQ2`)
   - Only WiFi devices supported (`NetworkType.WIFI`)
   - API errors mapped in `core_async.py::API2_ERRORS`

3. **Device state monitoring:**
   - `Monitor` class handles retries and auth refresh automatically
   - Max 10 consecutive failures before critical error (`MAX_UPDATE_FAIL_ALLOWED`)
   - Devices share a single `ClientAsync` via lock to prevent parallel auth refreshes

4. **Translation keys:**
   - Stored in `translations/*.json` with ISO 639-1 language codes
   - Config flow errors reference keys like `error_connect`, `invalid_credentials`

5. **Feature detection:**
   - Use `available_features` dict on device wrappers (e.g., `WashDeviceFeatures`, `AirConditionerFeatures`)
   - Features defined as `StrEnum` in `wideq/const.py`

## Debugging Tips

- Enable debug logging: Set logger level for `custom_components.smartthinq_sensors` and `custom_components.smartthinq_sensors.wideq`
- Check `device._api.device` for raw ThinQ device state
- Monitor auth issues via `Monitor._not_logged_count` and `_invalid_credential_count`
- Use `diagnostics.py` for device info dumps (includes model info, device status)

## Quick Reference: Key Files

- `__init__.py` - Integration entry point, `LGEDevice` coordinator class
- `config_flow.py` - Configuration UI flow
- `const.py` - Constants, feature enums, version info
- `wideq/core_async.py` - ThinQ API client (OAuth, requests)
- `wideq/device.py` - Base `Device` class and `Monitor` for state polling
- `wideq/factory.py` - Device instantiation factory
- `device_helpers.py` - Common formatting and device wrapper utilities
