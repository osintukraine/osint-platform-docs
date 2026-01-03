# Gazetteer Expansion

The location gazetteer is a local database of geographic locations used for offline geolocation of messages. By default, the platform includes Ukraine (UA) and Russia (RU) locations. This guide covers expanding the gazetteer to include additional countries.

## Overview

The gazetteer uses GeoNames data to match location names mentioned in messages to geographic coordinates. When messages mention places outside the default coverage (UA/RU), the geolocation pipeline falls back to slower LLM or Nominatim API lookups.

Expanding the gazetteer to include additional countries:
- Improves geolocation accuracy for referenced locations
- Reduces reliance on external APIs
- Enables proper handling of off-topic content (mapping Venezuela to Venezuela, not Ukraine)

## Quick Start

```bash
# Download and import Europe + USA locations
./scripts/expand_gazetteer.sh
```

## Manual Import

For specific countries, use the import script directly:

```bash
# Set database credentials
export POSTGRES_HOST=localhost
export POSTGRES_PORT=5432
export POSTGRES_DB=osint_platform
export POSTGRES_USER=osint_user
export POSTGRES_PASSWORD=your_password

# Download country data (e.g., Poland)
curl -sL "https://download.geonames.org/export/dump/PL.zip" -o data/geonames/PL.zip
unzip data/geonames/PL.zip -d data/geonames/

# Import to database
python scripts/import_geonames.py data/geonames/PL.txt
```

## Included Countries

The `expand_gazetteer.sh` script imports these countries:

| Code | Country | Locations | Reason |
|------|---------|-----------|--------|
| US | United States | ~2.2M | Aid/policy discussions |
| BY | Belarus | ~35K | Key neighbor, staging area |
| PL | Poland | ~58K | Transit/logistics hub |
| GB | United Kingdom | ~109K | Major ally |
| DE | Germany | ~214K | Major ally |
| FR | France | ~174K | Major ally |
| TR | Turkey | ~97K | Bosphorus, drones, diplomacy |
| RO | Romania | ~60K | NATO border |
| MD | Moldova | ~2.5K | Transnistria risk |
| GE | Georgia | ~9K | Russian occupation |
| LT | Lithuania | ~34K | Kaliningrad corridor |
| LV | Latvia | ~31K | Baltic state |
| EE | Estonia | ~15K | Baltic state |
| SK | Slovakia | ~11K | Neighbor |
| HU | Hungary | ~25K | Neighbor |
| CZ | Czech Republic | ~43K | Arms supplier |
| AT | Austria | ~53K | Neutral observer |
| FI | Finland | ~553K | NATO member, border |
| SE | Sweden | ~97K | NATO member |
| NO | Norway | ~608K | Arctic, NATO |
| IT | Italy | ~124K | G7 member |
| VE | Venezuela | ~69K | Off-topic detection |

## Check Gazetteer Status

```bash
# View locations by country
docker exec osint-postgres psql -U osint_user -d osint_platform -c "
SELECT country_code, COUNT(*) as locations
FROM location_gazetteer
GROUP BY country_code
ORDER BY locations DESC;"
```

## Data Source

Location data is sourced from [GeoNames](https://www.geonames.org/), licensed under Creative Commons Attribution 4.0.

Files downloaded from: `https://download.geonames.org/export/dump/{CC}.zip`

## Troubleshooting

### Import Fails with Database Error

If import fails partway through:

```bash
# Check how many locations were imported
docker exec osint-postgres psql -U osint_user -d osint_platform -c "
SELECT country_code, COUNT(*) FROM location_gazetteer WHERE country_code = 'GB' GROUP BY country_code;"

# Re-run import (duplicate entries are skipped)
python scripts/import_geonames.py data/geonames/GB.txt
```

### Large Country Imports Slow

Countries with >500K locations (US, FI, NO) can take 10-15 minutes. Run in background:

```bash
nohup python scripts/import_geonames.py data/geonames/US.txt > import_us.log 2>&1 &
```

### Missing Country Data

Download from GeoNames directly:

```bash
curl -sL "https://download.geonames.org/export/dump/{CC}.zip" -o /tmp/{CC}.zip
unzip /tmp/{CC}.zip -d data/geonames/
```

## Related

- [Event Detection V3](../architecture/event-detection-v3.md) - Geolocation pipeline architecture
- [Off-Topic Detection](../features/off-topic-detection.md) - How off-topic content is handled
