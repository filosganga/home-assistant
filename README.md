# home-assistant

Home Assistant and a Mosquitto broker, running in Docker on a Raspberry Pi 5.

This repository is the machine's configuration. Everything here is tracked so
the box can be rebuilt; everything secret is not — see `.gitignore`, which
covers `mosquitto/config/passwd`, `ha/config/secrets.yaml`, `.storage/` and
`.env`.

## Layout

The compose file maps two directories, and the nesting is easy to get wrong:

| On disk | In the container |
|---|---|
| `./ha/config` | Home Assistant's `/config` |
| `./mosquitto/config` | `/mosquitto/config` |
| `./mosquitto/data` | `/mosquitto/data` |

So Home Assistant's configuration is **`ha/config/`**, not `config/`. Checked
out at `~/ha` on the Pi, that is `~/ha/ha/config/`.

`ha/config` is root-owned, because the container writes there as root. Editing
on the Pi needs `sudo`, which is why changes belong here and arrive by `git
pull` rather than being typed on the box.

## Running it

```sh
docker compose up -d
docker compose logs -f homeassistant
docker compose logs -f mosquitto      # connects, disconnects, auth failures
```

Home Assistant runs with `network_mode: host`. That has one consequence worth
knowing before debugging anything: **it reaches Mosquitto at `localhost`, not
at `mosquitto`.** A container name only resolves on a bridge network, and this
service is not on one.

## Mosquitto

`mosquitto/config/mosquitto.conf`:

```
listener 1883
allow_anonymous false
password_file /mosquitto/config/passwd
persistence true
```

`persistence true` is load-bearing rather than tidy. Everything that remembers
state in the cat-feeder system is a retained message, and without persistence
they are lost on every broker restart. Most come straight back — the package
below republishes the time each minute and the schedule on start — but a
feeder's `paused` flag does not, and a paused feeder silently resuming is the
one state change nothing alarms about.

### Users

The password file is not tracked. Two users exist:

| User | For |
|---|---|
| `ha` | Home Assistant's own MQTT integration |
| `cat-feeder` | the feeders |

⚠️ **Add a user with `mosquitto_passwd -b`, never `-c`.** The `-c` flag
*creates* the file, which means it truncates: it would delete both users, and
Home Assistant would lose its broker connection along with them.

```sh
docker exec -it mosquitto mosquitto_passwd -b /mosquitto/config/passwd <user> <password>
docker restart mosquitto
```

## Home Assistant

Onboarded through the UI. Two settings from that are worth recording because
nothing in this repo captures them — they live in `.storage/`, which is
deliberately untracked:

- **Timezone `Europe/Rome`.** This is not cosmetic. The cat feeders read the
  wall-clock fields of the time this instance publishes and do not convert
  them, so an instance on UTC publishes a payload that is entirely valid with
  every mealtime moved by the offset. The offset on the wire is the only thing
  that proves it.
- **The MQTT integration**, pointing at `localhost:1883` as user `ha`.

A rebuild has to redo both by hand.

## The cat feeders

Three ESP32-C6 feeders take their orders from this instance over MQTT. Their
firmware lives in a separate repository and knows nothing about this machine —
no address, no hostname, no path — which is why pointing a feeder here is a
flag rather than a rebuild.

What this side owes them is **`ha/config/packages/cat_feeder.yaml`**, copied
unchanged from that repository's `homeassistant/packages/`. It publishes the
time every minute, publishes the schedule, and pauses the feeders when the
`cat_feeder_active` helper goes off.

Installing it takes two things, and the second is easy to miss:

```yaml
# ha/config/configuration.yaml
homeassistant:
  packages: !include_dir_named packages
```

An instance set up through the UI has no `homeassistant:` block at all, and
without one the packages directory is never read — the file loads cleanly and
does nothing.

⚠️ **Install this before pointing any feeder here.** The feeders have no clock
of their own. One pointed at a broker where nobody publishes the time takes the
*retained* time, cannot tell how old it is, and therefore refuses to arm its
schedule: it sits online, flashing red three times, and never feeds. That is
correct behaviour and indistinguishable from a fault.

Check it with no feeder involved:

```sh
mosquitto_sub -h localhost -u cat-feeder -P <password> -v -t 'feeder/time'
```

A line a minute means this side is done. Read the offset on it before believing
it.

## Rebuilding

1. Clone this repo to `~/ha`.
2. Create `mosquitto/config/passwd` with the two users above.
3. `docker compose up -d`.
4. Onboard Home Assistant — **set the timezone to `Europe/Rome`**.
5. Add the MQTT integration at `localhost:1883` as `ha`.
6. Copy `cat_feeder.yaml` into `ha/config/packages/` and confirm the
   `homeassistant: packages:` include is present.
7. Verify `feeder/time` is publishing before repointing any feeder.
