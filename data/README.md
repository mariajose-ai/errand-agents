# Data

All experiments run offline against frozen files in this folder. Nothing is fetched
during a run.

## Layout

| Path | What it is | In git? |
|---|---|---|
| `raw/` | Original OSM downloads | No (large; see .gitignore) |
| `osm/` | Frozen, filtered extract: the connected pedestrian subgraph and candidate venues | Yes |
| `venues.csv` | Hand-curated hours and services for 25–30 verified venues, one source link per row | Yes |
| `world.json` | The simulator world built from `osm/` + `venues.csv` | Yes |
| `toy_world.json` | Made-up world for tests and development only; not real places | Yes |
| `scenarios/` | Generated scenario sets (JSONL), with the seed used | Yes |

## Sources

### OpenStreetMap
- **Area:** Stanford campus and nearby Palo Alto (bounding box: TBD)
- **Snapshot date:** TBD
- **Downloaded from:** TBD (e.g. Overpass API query, saved in `osm/query.overpassql`)
- **Used for:** pedestrian paths and amenity locations

### Curated venue details
OSM opening hours and services are often missing or out of date, so each venue's hours
and services are checked against a public source by hand. `venues.csv` records, for each
venue: OSM id, name, services, hours, source URL, and the date it was checked.

- **Curation date(s):** TBD
- **Rule:** only venues whose hours and services can be verified are included; no
  invented locations.

## License and attribution

Map data © OpenStreetMap contributors, available under the
[Open Database License (ODbL)](https://www.openstreetmap.org/copyright).

- Any public use (plots, maps, the write-up) must credit "© OpenStreetMap contributors".
- If the derived extract in `osm/` or `world.json` is shared publicly, it must also be
  offered under the ODbL.

The data contains no personal information.
