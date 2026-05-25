---
title: "Readsb ADS-B Aircraft State Archive"
date: 2026-05-25
lastmod: 2026-05-25
description: "Create a local DuckDB database from your ADS-B decoder data."
tags: ["ads-b", "readsb"]
categories: ["Air traffic data"]
draft: false

# --- Hero image ---
# Drop any image named feature.jpg / feature.png / feature.webp into this
# directory. It becomes the card thumbnail, the hero banner, AND the
# Open Graph image shared on Twitter/LinkedIn/Slack automatically.
showHero: true
heroStyle: "big"          # basic | big | background | thumbAndBackground

# --- Reading experience ---
showTableOfContents: true
showReadingTime: true
showDateUpdated: true
showTaxonomies: true
showPagination: true

# --- dev.to cross-posting ---
# Publish here first, get your canonical URL, then paste it into the
# dev.to editor under "Advanced" → "Canonical URL". That's all you need.
# Google will index YOUR site as the original source.
---

<!-- Replace everything below with your real content. -->

## Introduction

If you are feeding air traffic data to any of the ADS-B/ModeS aggregators, this post will show you how to create a local historical aircraft state dataset that you can query with DuckDB using Bento and Parquet files.

## Prerequisites 

The following assumes:
-  You are already feeding ADS-B/ModeS data to any of the aggregators using a [readsb](https://github.com/wiedehopf/readsb) based decoder/feeder which you are running on a Raspberry Pi or other Linux OS.
- You are comfortable with Docker containers and Docker Compose and basic Linux commands.
- The following was tested with [docker-adsb-ultrafeeder](https://github.com/sdr-enthusiasts/docker-adsb-ultrafeeder) as the ADS-B data collector which many enthusiasts are now using to feed to all of the aggregators. 


## Readsb 

Readsb is the Swiss Army Knife of ADS-B/ModeS decoding and has many features and config options. For this project we are going to use its aircraft state TCP output stream. Readsb emits an aircraft "state" JSON object per aircraft every time a new position is received. This object contains a lot of information, below is an example output: 

```JSON
{
  "now": 1779679841.113,
  "hex": "45202b",
  "type": "adsb_icao",
  "flight": "CGF1808 ",
  "alt_baro": 15350,
  "alt_geom": 16425,
  "gs": 355.4,
  "ias": 280,
  "tas": 358,
  "mach": 0.556,
  "wd": 25,
  "ws": 18,
  "oat": 0,
  "tat": 17,
  "track": 111.8,
  "track_rate": -0.03,
  "roll": -1.76,
  "mag_heading": 111.8,
  "true_heading": 107.95,
  "baro_rate": 2944,
  "geom_rate": 2912,
  "squawk": "7706",
  "emergency": "none",
  "category": "A0",
  "nav_qnh": 1013.6,
  "nav_altitude_mcp": 24992,
  "nav_heading": 111.8,
  "lat": 50.721863,
  "lon": 7.788267,
  "nic": 8,
  "rc": 186,
  "seen_pos": 0,
  "version": 2,
  "nic_baro": 1,
  "nac_p": 10,
  "nac_v": 0,
  "sil": 3,
  "sil_type": "perhour",
  "gva": 2,
  "sda": 2,
  "alert": 0,
  "spi": 0,
  "mlat": [],
  "tisb": [],
  "messages": 161,
  "seen": 0,
  "rssi": -49.5
}
```

The [ultrafeeder](https://github.com/sdr-enthusiasts/docker-adsb-ultrafeeder) should have this feature enabled and exposed via the default port `30047`. To test this is working copy and paste the following into your commandline: 

```bash
nc localhost 30047 | jq --unbuffered -c '.'
```
You should see a bunch of JSON objects written to the terminal like the above example. If not, search in the [documentation](https://github.com/sdr-enthusiasts/docker-adsb-ultrafeeder) for how to enable this feature with flag `--net-json-port=`, or double check your Docker port settings. 

## Bento
[Bento](https://warpstreamlabs.github.io/bento/) is an open source stream processing binary written in Go that makes common data engineering tasks very simple to implement. In-short it connects to a `source`, allows you to do `processing` in-between flushing the results out to an `output`. 

Check out the [documentation](https://warpstreamlabs.github.io/bento/) for more info. 

## Let's do this...
This is a diagram of what we are trying to achieve: 
```bash
ultrafeeder :30047 (JSON stream)
    └── Bento (batch → Parquet encode)
            └── /data/bento/aircraft/hour=.../part-N.parquet
                    └── DuckDB view
```
### Install dependencies
On your Pi running the ultrafeeder:
1) Create the data directory which will store the output from readsb.
```bash
sudo mkdir -p /data/bento/aircraft
sudo chown 10001:10001 /data/bento/aircraft
```
2) Download this Bento config file (full repo is [here](https://github.com/johnfitzy/readsb-state-archive))
```bash
# curl
curl -O https://raw.githubusercontent.com/johnfitzy/readsb-state-archive/main/bento-config.yaml

# or wget
wget https://raw.githubusercontent.com/johnfitzy/readsb-state-archive/main/bento-config.yaml
```
3) Run the Bento container
- Note the volume mapping to the directory we created earlier
- Also see the config file you just downloaded `bento-config.yaml` mapped to `/bento.yaml` in the container. 
```bash
docker run --net=host -d --restart unless-stopped \
  --name readsb-state-archive \
  -v $(pwd)/bento-config.yaml:/bento.yaml \
  -v /data/bento/aircraft:/data/bento/aircraft \
  ghcr.io/warpstreamlabs/bento:1.17.0
```
## Wait

Bento will start collecting immediately. By default it uses a 15-minute tumbling window, your first Parquet file will appear after the first window closes (15min intervals past the hour). This is configured with the `buffer.system_window.size` option in `bento-config.yaml`.

## Verify output
After 15 minutes, check for Parquet files:
```bash
ls /data/bento/aircraft/
```
## DuckDB

1) Install DuckDB
#### Raspberry Pi (aarch64):
```bash
wget https://github.com/duckdb/duckdb/releases/latest/download/duckdb_cli-linux-arm64.zip
unzip duckdb_cli-linux-arm64.zip
sudo mv duckdb /usr/local/bin/
```
#### x86-64 (amd64):
```bash
wget https://github.com/duckdb/duckdb/releases/latest/download/duckdb_cli-linux-amd64.zip
unzip duckdb_cli-linux-amd64.zip
sudo mv duckdb /usr/local/bin/
```
2) Once you see the first Parquet file in the data directory (`ls /data/bento/aircraft/`), continue to the next step.

## Create a persistent view
Create a DuckDB database file once, it stores only the view definition, not the data:

```bash
duckdb ~/aircraft.duckdb
```
```bash
CREATE VIEW aircraft AS
SELECT * FROM read_parquet('/data/bento/aircraft/hour=*/part-*.parquet', hive_partitioning=true);
```
Reconnecting later with duckdb ~/aircraft.duckdb will have the view ready to go.

### Example queries
```bash
-- How many distinct aircraft seen?
SELECT COUNT(DISTINCT hex) FROM aircraft;

-- All flights in the last hour
SELECT DISTINCT hex, flight, alt_baro_ft, gs
FROM aircraft
WHERE time > epoch(now()) - 3600
ORDER BY flight;

-- Highest aircraft seen
SELECT hex, flight, MAX(alt_baro_ft) AS max_alt
FROM aircraft
GROUP BY hex, flight
ORDER BY max_alt DESC
LIMIT 20;

-- Aircraft count by hour
SELECT hour, COUNT(DISTINCT hex) AS aircraft
FROM aircraft
GROUP BY hour
ORDER BY hour DESC;
```

## Disk use
**IMPORTANT:** Be sure to rotate the files (or mount the directory to your NAS) because eventually your Pi's disk will fill up!


## Conclusion
Now you have your own local data pipeline which is saving the ADS-B data you collect to Parquet files that you can run SQL queries on with DuckDB. 

## Extra
Normally, in the "real" world, Parquet file size should be kept between 256MB to 1GB for best performance. With your own ADS-B data feed you won't get any where near this size for each hour partition. To increase the file size, increase the tumbling window up to 1h. But make sure you are not going to blow the Pi's memory out, eg: docker stats to see RAM usage of the container.
