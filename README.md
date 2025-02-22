# TeslaMate

[![](https://travis-ci.org/adriankumpf/teslamate.svg?branch=master)](https://travis-ci.org/adriankumpf/teslamate)
[![](https://coveralls.io/repos/github/adriankumpf/teslamate/badge.svg?branch=master)](https://coveralls.io/github/adriankumpf/teslamate?branch=master)
[![](https://images.microbadger.com/badges/version/teslamate/teslamate.svg)](https://hub.docker.com/r/teslamate/teslamate)
[![](https://images.microbadger.com/badges/image/teslamate/teslamate.svg)](https://microbadger.com/images/teslamate/teslamate)
[![](https://img.shields.io/docker/pulls/teslamate/teslamate?color=%23099cec)](https://hub.docker.com/r/teslamate/teslamate)
[![](https://img.shields.io/badge/Donate-PayPal-ff69b4.svg)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=YE4CPXRAV9CVL&source=url)

A data logger for your Tesla.

- Written in [Elixir](https://elixir-lang.org/)
- Data is stored in PostgreSQL
- Visualization and data analysis with Grafana
- Current vehicle data is published to a local MQTT Broker _(optional)_

## Features

**Dashboards**

- Lifetime driving map
- Drive and charging reports
- Driving efficiency report
- Consumption (net / gross)
- Vampire drain
- Projected 100% range (battery degradation)
- SOC charging stats
- Visited addresses
- History of installed updates

**General**

- Little to no additional vampire drain: the car will fall asleep after a
  certain idle time
- Built-in API to manually suspend / resume sending requests to the Tesla API
- Automatic address lookup
- Locally enriches positions with elevation data
- Geo-fencing feature to create custom locations
- Supports multiple vehicles per Tesla Account

## Screenshots

![Drive Details](screenshots/drive.png)
![Drives](screenshots/drives.png)
![Efficiency](screenshots/efficiency.png)
![States](screenshots/states.png)
![Charge Details](screenshots/charging.png)
![Charges](screenshots/charging_history.png)
![Vampire Drain](screenshots/vampire_drain.png)

## Table of contents

- [Installation](#installation)
- [Configuration](#configuration)
- [Upgrading](#upgrading)
- [Web Interface](#web-interface)
- [MQTT](#mqtt)
- [FAQ](#faq)
- [Contributing](#contributing)
- [Donations](#donations)
- [Disclaimer](#disclaimer)
- [Credits](#credits)

---

## Installation

### Docker (recommended)

The recommended way to install and run TeslaMate is to use Docker. Create a
`docker-compose.yml` file and replace the necessary `environment` variables:

```YAML
version: '3'
services:
  teslamate:
    image: teslamate/teslamate:latest
    restart: unless-stopped
    environment:
      - DATABASE_USER=teslamate
      - DATABASE_PASS=secret
      - DATABASE_NAME=teslamate
      - DATABASE_HOST=db
      - MQTT_HOST=mosquitto
      - VIRTUAL_HOST=localhost # if you're going to access the UI from another  machine replace
                               # "localhost" with the hostname / IP address of the docker host.
      - TZ=Europe/Berlin       # (optional) replace to use local time in debug logs. See "Configuration".
    ports:
      - 4000:4000
    cap_drop:
      - all

  db:
    image: postgres:11
    environment:
      - POSTGRES_USER=teslamate
      - POSTGRES_PASSWORD=secret
    volumes:
      - teslamate-db:/var/lib/postgresql/data

  grafana:
    image: teslamate/grafana:latest
    environment:
      - DATABASE_USER=teslamate
      - DATABASE_PASS=secret
      - DATABASE_NAME=teslamate
      - DATABASE_HOST=db
    ports:
      - 3000:3000
    volumes:
      - teslamate-grafana-data:/var/lib/grafana

  mosquitto:
    image: eclipse-mosquitto:1.6
    ports:
      - 1883:1883
    volumes:
      - mosquitto-conf:/mosquitto/config
      - mosquitto-data:/mosquitto/data

volumes:
    teslamate-db:
    teslamate-grafana-data:
    mosquitto-conf:
    mosquitto-data:
```

Afterwards start everything with `docker-compose up`.

Open the web interface at http://ip-of-your-machine:4000 and sign in with your
Tesla Account.

To access Grafana go to http://ip-of-your-machine:3000.

**Optional:** To switch to **imperial measurements** open the web interface and
navigate to `Settings`.

_For a more advanced setup check out the wiki: [Advanved Setup (SSL, FQDN, pw
protected)](<https://github.com/adriankumpf/teslamate/wiki/Advanved-Setup-(SSL,-FQDN,-pw-protected)>)_

### Manual Installation (advanced)

To run TeslaMate without Docker follow this guide: [Manual
Installation](https://github.com/adriankumpf/teslamate/wiki/Manual-Installation)

## Configuration

TeslaMate uses environment variables for runtime configuration.

### Environment Variables

| Variable Name                 | Description                                                                                                                                                                                                                                                                                                                          | Default Value |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------- |
| DATABASE_USER                 | Username (**required**)                                                                                                                                                                                                                                                                                                              | /             |
| DATABASE_PASS                 | User password (**required**)                                                                                                                                                                                                                                                                                                         | /             |
| DATABASE_NAME                 | The database to connect to (**required**)                                                                                                                                                                                                                                                                                            | /             |
| DATABASE_HOST                 | Hostname of the database server (**required**)                                                                                                                                                                                                                                                                                       | /             |
| DATABASE_PORT                 | Port of the database server                                                                                                                                                                                                                                                                                                          | 5432          |
| DATABASE_POOL_SIZE            | Size of the database connection pool                                                                                                                                                                                                                                                                                                 | 5             |
| VIRTUAL_HOST                  | Host part used for generating URLs throughout the app                                                                                                                                                                                                                                                                                | localhost     |
| CHECK_ORIGIN                  | Configures whether to check the origin header or not _(optional)_. May be `true` (default), `false` or a comma-separated list of hosts that are allowed (e.g. `https://example.com,//another.com:888`). Hosts also support wildcards. It defaults to true and, in such case, it will check against the host value in `VIRTUAL_HOST`. | true          |
| PORT                          | Port where the web interface is exposed                                                                                                                                                                                                                                                                                              | 4000          |
| DISABLE_MQTT                  | Disables the MQTT feature if `true`                                                                                                                                                                                                                                                                                                  | false         |
| MQTT_HOST                     | Hostname of the broker (**required** unless DISABLE_MQTT is `true`)                                                                                                                                                                                                                                                                  | /             |
| MQTT_USERNAME                 | Username _(optional)_                                                                                                                                                                                                                                                                                                                | /             |
| MQTT_PASSWORD                 | Password _(optional)_                                                                                                                                                                                                                                                                                                                | /             |
| MQTT_TLS                      | Enables TLS if `true` _(optional)_                                                                                                                                                                                                                                                                                                   | false         |
| MQTT_TLS_ACCEPT_INVALID_CERTS | Accepts invalid certificates if `true` _(optional)_                                                                                                                                                                                                                                                                                  | false         |
| LOCALE                        | The default locale for the web interface and addresses. Currently available: `en` (default) and `de`                                                                                                                                                                                                                                 | en            |
| TZ                            | Used to establish the local time zone. See [List of tz database time zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).                                                                                                                                                                                           | /             |

## Upgrading

> Check the [Changelog](CHANGELOG.md) before upgrading!

### Docker

Run `docker-compose pull` and restart with `docker-compose up`.

### Manually

See [Manual Installation](https://github.com/adriankumpf/teslamate/wiki/Manual-Installation#upgrading)

## Web Interface

There is a basic real-time web interface. Once the application is running locally, you
can access it at [localhost:4000](http://localhost:4000).

![Web Interface](screenshots/web_interface.png)

## MQTT

Unless the MQTT feature is disabled data is published to the following topics
(`$car_id` usually starts at 1):

```bash
teslamate/cars/$car_id/display_name
teslamate/cars/$car_id/state
teslamate/cars/$car_id/healthy

teslamate/cars/$car_id/latitude
teslamate/cars/$car_id/longitude
teslamate/cars/$car_id/shift_state
teslamate/cars/$car_id/speed # km/h

teslamate/cars/$car_id/locked
teslamate/cars/$car_id/sentry_mode
teslamate/cars/$car_id/windows_open

teslamate/cars/$car_id/outside_temp # °C
teslamate/cars/$car_id/inside_temp # °C

teslamate/cars/$car_id/odometer # km
teslamate/cars/$car_id/ideal_battery_range_km
teslamate/cars/$car_id/est_battery_range_km
teslamate/cars/$car_id/rated_battery_range_km

teslamate/cars/$car_id/battery_level
teslamate/cars/$car_id/plugged_in
teslamate/cars/$car_id/charge_energy_added
teslamate/cars/$car_id/charge_limit_soc
teslamate/cars/$car_id/charge_port_door_open
teslamate/cars/$car_id/charger_actual_current
teslamate/cars/$car_id/charger_phases
teslamate/cars/$car_id/charger_power
teslamate/cars/$car_id/charger_voltage
teslamate/cars/$car_id/scheduled_charging_start_time
teslamate/cars/$car_id/time_to_full_charge
```

## FAQ

### Sometimes the first few minutes of a drive are not recorded even though the car was online. Why?

TeslaMate polls the car every few seconds while driving or charging. After
that, it keeps polling for about 15 minutes (to catch the following drive if
you stopped for a drink, for example). After this period, TeslaMate will stop
polling the car for about 21 minutes to let it go to sleep. This is repeated
until the car is asleep or starts to drive/charge again. Once sleeping,
TeslaMate will never wake the wake the car. It is only checked twice a minute to
see if it is still sleeping.

This approach may sometimes lead to _small_ data gaps: if the car starts
driving during the 21 minute period where TeslaMate is not polling, nothing can
be logged.

##### Solution

To get around this you can use your smartphone to inform TeslaMate when to
start polling again. In short, create a workflow with
[Tasker](https://play.google.com/store/apps/details?id=net.dinglisch.android.taskerm&hl=en)
(Android) or [Shortcuts](https://support.apple.com/guide/shortcuts/welcome/ios)
(iOS) that listens for connected Bluetooth devices. If a connection to your
Tesla is established send an HTTP PUT `resume` request to your publicly exposed
TeslaMate instance. See the available commands below.

**Alternatively** / additionally, you can experiment with the sleep settings.
Some cars, especially Model 3, seem to handle a `Time to Try Sleeping` value of
12 min just fine. Doing so reduces the likelihood of potential data gaps. Just
keep an eye on your car afterwards to see if it still goes into sleep mode.

##### Available Commands

```
PUT https://teslamate.your-domain.com/api/car/$car_id/logging/resume
PUT https://teslamate.your-domain.com/api/car/$car_id/logging/suspend
```

⚠️ I strongly recommend to use a reverse-proxy with HTTPS and basic access
authentication when exposing TeslaMate to the public internet. Additionally
only permit access to `/api/car/$car_id/logging/resume` and/or
`/api/car/$car_id/logging/suspend`. See [Advanved Setup (SSL, FQDN, pw
protected)](<https://github.com/adriankumpf/teslamate/wiki/Advanved-Setup-(SSL,-FQDN,-pw-protected)>).

### My car is not sleeping!

Please follow the steps applicable to TeslaMate mentioned in [this
article](https://support.teslafi.com/knowledge-bases/2/articles/161-my-vehicle-is-not-sleeping).

Most importantly, if you have a Model S or X built prior to March 2018 make
sure 'Always Connected' is turned off and 'Energy Savings' is turned on in the
vehicle.

### What is the geo-fence feature for?

At the moment geo-fences are a way to create custom locations like `🏡 Home` or
`🛠️ Work` That may be particularly useful if the addresses (which are provided
by [OSM](https://www.openstreetmap.org)) in your region are inaccurate or if
you street-park at locations where the exact address may vary.

## Contributing

All contributions are welcome and greatly appreciated!

### Initial Setup

To run the TeslaMate test suite you need PostgreSQL 10+ running locally with a
database named `teslamate_test`. Follow these steps to create the database and
run all migrations:

```bash
# download dependencies
mix deps.get

# create the dev database and run migrations
mix ecto.setup

# create the test database
MIX_ENV=test mix ecto.setup

# download JavaScript dependencies
npm install --prefix ./assets
```

A local MQTT broker like `mosquitto` is not required but recommend.

### Development

Start an iex session in another terminal window:

```elixir
iex -S mix phx.server
```

Then sign in with a Tesla account.

**Hot reloading**

To immediately apply your local changes open or reload
[http://localhost:4000](http://localhost:4000). You can also reload specific
modules via `iex`, for example:

```elixir
iex> r TeslaMate.Vehicles.Vehicle
```

To only compile the changes:

```bash
mix compile
```

**Code formatting**

```bash
mix format
```

### Testing

To ensure a commit passes CI you should run `mix ci` locally, which executes the following commands:

- Check formatting (`mix format --check-formatted`)
- Run all tests (`mix test`)

### Making Changes to Grafana Dashboards

To update dashboards you need Grafana running locally. The following
_docker-compose.yml_ can be used for this purpose:

```YAML
version: '3'
services:
  grafana:
    image: teslamate-grafana:latest
    environment:
      - DATABASE_USER=postgres
      - DATABASE_PASS=postgres
      - DATABASE_NAME=teslamate_dev
      - DATABASE_HOST=host.docker.internal
    ports:
      - 3000:3000
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
    grafana-data:
```

_(on Linux use the actual IP address of the host as `DATABASE_HOST`instead of
`host.docker.internal`)_

Then build the image with `make grafana` and run the container via
`docker-compose up grafana`.

Access the Grafana at [http://localhost:3000](http://localhost:3000) and sign
in with the default user `admin` and password `admin`.

Then edit the respective dashboard(s) locally. To export a dashboard hit the
'Save' button and select `Save JSON to file`. The final JSON file belongs in the
directory `./grafana/dashboards/`. To apply the changes rebuild the image and
start the container.

## Donations

Maintaining this project isn't effortless, or free. If you would like to kick
in and help me cover those costs, that would be awesome. If you don't, no
problem; just share your love and show your support.

<p align="center">
  <a href="https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=YE4CPXRAV9CVL&source=url">
    <img src="screenshots/paypal-donate-button.png" alt="Donate with PayPal" />
  </a>
</p>

## Disclaimer

Please note that the use of the Tesla API in general and this software in
particular is not endorsed by Tesla. Use at your own risk.

## Credits

[TeslaLogger](https://github.com/bassmaster187/TeslaLogger) was a big
inspiration especially during early development. Thanks!
