# PlantPal Data Schema

This document is the canonical human-readable data dictionary for the `PlantPal-Data` repository.

Its goal is simple:

- let a human understand the data layout quickly
- let an AI agent read one file and safely work with the repo
- keep the JSON storage stable even as the Android app and backend evolve

For now, this document is the source of truth. If we later want strict machine validation, we should generate separate JSON Schema files from this design rather than replacing it with one giant `schema.json`.

## Design Goals

- offline-first and Git-friendly
- easy to diff in GitHub
- minimal duplication
- append-only where practical
- compatible with a private repo used as the shared source of truth

## Repository Layout

Current and planned layout:

```text
/schema.md
/readme.md
/data
  /zones
    /{zone-slug}
      zone.json
  /plants
    /{species-slug}
      /{plant-slug}
        plant.json
        /observations
          {timestamp}--{observationId}.json
        /pictures
          {timestamp}--{pictureId}.jpg
          {timestamp}--{pictureId}.picture.json
```

Notes:

- This repo is plant-centric. Each plant has its own folder containing the main record, its observations, and its pictures.
- Zones are few, stable, and stored in their own folders.
- Image bytes are stored as files under each plant's `pictures/` folder.
- Image metadata is stored in sidecar `.picture.json` files next to the image.
- Image bytes should not be embedded inline in JSON except for temporary migration/debug scenarios.

## JSON Conventions

These rules apply to every record type:

- File encoding: UTF-8
- Dates and times: ISO 8601 UTC strings, example: `2026-03-14T12:34:56Z`
- IDs: lowercase GUID strings
- Soft delete: use `deletedAtUtc: null` for active records; set it to a UTC timestamp when deleted
- Unknown optional values: use `null`, not empty objects
- Relationships: reference other records by ID; do not embed full related records
- Derived values should not be persisted unless explicitly called out

## File-Level Storage Rules

This repo uses per-record and per-plant files instead of giant collection arrays.

Rules:

- `zone.json` stores one zone record
- `plant.json` stores one plant record
- each observation is stored in its own file under `observations/`
- each picture file may have a sidecar metadata file ending in `.picture.json`

Why this structure:

- easier for a human to browse
- lower Git churn
- more atomic changes
- plant records can be loaded and saved as a unit
- observations remain append-only

Slug rules:

- `zone-slug` should be a stable human-readable slug such as `turmeric-grove`
- `species-slug` should usually come from the scientific name, such as `althaea-officinalis`
- `plant-slug` should be human-readable and unique within the species folder
- if two plants would collide, add a short suffix, for example `marshmallow-patch--9d85d070`

Identity rules:

- path names are for browsing convenience
- JSON `id` values are the authoritative identity
- code must never infer record identity only from the folder name

## Common Record Metadata

Every persisted entity should include these fields unless explicitly noted otherwise:

| Field | Type | Required | Meaning |
| --- | --- | --- | --- |
| `id` | string (GUID) | yes | stable primary key |
| `createdAtUtc` | string | yes | record creation time |
| `updatedAtUtc` | string | yes | last modification time |
| `deletedAtUtc` | string or null | yes | soft-delete marker |
| `version` | integer | yes | increment on every write |
| `sourceDeviceId` | string or null | no | device or process that last created/updated the record |

Rules:

- `updatedAtUtc` must be greater than or equal to `createdAtUtc`
- `version` starts at `1`
- soft-deleted records stay in the file until a later compaction/archive workflow exists

## Entity: Zone

Zones represent physical areas, beds, patches, containers, or logical growing regions.

Stored in:

- `data/zones/{zone-slug}/zone.json`

Required fields:

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | string | zone ID |
| `name` | string | display name |
| `createdAtUtc` | string | metadata |
| `updatedAtUtc` | string | metadata |
| `deletedAtUtc` | string or null | metadata |
| `version` | integer | metadata |

Optional fields:

| Field | Type | Meaning |
| --- | --- | --- |
| `description` | string or null | freeform notes about the zone |
| `parentZoneId` | string or null | supports nested zones like `Back Yard > Turmeric Bed` |
| `latitude` | number or null | representative latitude for the zone center |
| `longitude` | number or null | representative longitude for the zone center |
| `geoRadiusMeters` | number or null | approximate radius for the zone |
| `sunExposure` | string or null | suggested values: `full_sun`, `partial_shade`, `shade`, `mixed` |
| `soilNotes` | string or null | summary notes about the zone's soil |
| `tags` | array of strings | optional labels |
| `sourceDeviceId` | string or null | metadata |

Derived data not stored in the zone record:

- list of plants in the zone
- list of pictures in the zone

Those are derived by scanning plant folders and picture metadata files.

Example:

```json
{
  "id": "3f7bb3a1-7d6d-4a7d-9a54-0f2ca0ed130f",
  "name": "Turmeric Grove",
  "description": "Moist, partially shaded bed near the east fence.",
  "parentZoneId": null,
  "latitude": 30.123456,
  "longitude": -97.123456,
  "geoRadiusMeters": 9.5,
  "sunExposure": "partial_shade",
  "soilNotes": "Dark soil with high organic matter.",
  "tags": ["east-yard", "shade-tolerant"],
  "createdAtUtc": "2026-03-14T12:00:00Z",
  "updatedAtUtc": "2026-03-14T12:00:00Z",
  "deletedAtUtc": null,
  "version": 1,
  "sourceDeviceId": "android-field-phone"
}
```

## Entity: Plant

Plants are the main catalog records. A plant should exist once and then accumulate observations over time.

Stored in:

- `data/plants/{species-slug}/{plant-slug}/plant.json`

Required fields:

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | string | plant ID |
| `name` | string | primary display name |
| `createdAtUtc` | string | metadata |
| `updatedAtUtc` | string | metadata |
| `deletedAtUtc` | string or null | metadata |
| `version` | integer | metadata |

Recommended optional fields:

| Field | Type | Meaning |
| --- | --- | --- |
| `zoneId` | string or null | owning zone |
| `commonName` | string or null | alternate common name |
| `scientificName` | string or null | botanical name |
| `description` | string or null | general description |
| `notes` | string or null | persistent notes |
| `plantedDate` | string or null | planting date if known |
| `status` | string | suggested values: `active`, `dormant`, `harvested`, `dead`, `archived` |
| `lifeCycle` | string or null | suggested values: `annual`, `perennial`, `biennial`, `unknown` |
| `environment` | string or null | suggested values: `indoor`, `outdoor`, `greenhouse`, `mixed`, `unknown` |
| `latitude` | number or null | precise plant coordinate if known |
| `longitude` | number or null | precise plant coordinate if known |
| `gpsAccuracyMeters` | number or null | GPS accuracy for the stored point |
| `primaryPictureId` | string or null | preferred display picture |
| `tags` | array of strings | labels like `medicinal`, `edible`, `native` |
| `careEvents` | array of CareEvent | recurring care rules for this plant |
| `sourceDeviceId` | string or null | metadata |

Important rules:

- `zoneId` is the relationship field; do not embed a full zone object in the plant record.
- If either `latitude` or `longitude` is present, both should be present.
- `primaryPictureId` must refer to a picture sidecar metadata record in the plant's `pictures/` folder.
- `careEvents` is embedded because it belongs tightly to the plant and is usually edited as part of the plant itself.

Example:

```json
{
  "id": "9d85d070-a627-4e95-8d3d-98e37cb3e5ef",
  "name": "Marshmallow Patch",
  "commonName": "Marshmallow",
  "scientificName": "Althaea officinalis",
  "zoneId": "3f7bb3a1-7d6d-4a7d-9a54-0f2ca0ed130f",
  "description": "Medicinal marshmallow cluster near the fence line.",
  "notes": "Likes moisture and afternoon protection from harsh sun.",
  "plantedDate": "2026-02-20T00:00:00Z",
  "status": "active",
  "lifeCycle": "perennial",
  "environment": "outdoor",
  "latitude": 30.123401,
  "longitude": -97.123510,
  "gpsAccuracyMeters": 2.4,
  "primaryPictureId": "a9cf3d78-1c5d-4976-9b0d-ef60771edca7",
  "tags": ["medicinal", "fence-line"],
  "careEvents": [
    {
      "id": "bfffb32b-b28c-4bb9-8f6d-f73f0aa9eb12",
      "type": "water",
      "intervalDays": 3,
      "notes": "Skip if soil is still wet after rain.",
      "isActive": true
    }
  ],
  "createdAtUtc": "2026-03-14T12:10:00Z",
  "updatedAtUtc": "2026-03-14T12:10:00Z",
  "deletedAtUtc": null,
  "version": 1,
  "sourceDeviceId": "android-field-phone"
}
```

## Embedded Type: CareEvent

Care events are recurring care rules attached directly to a plant.

Stored inside:

- `Plant.careEvents[]`

Fields:

| Field | Type | Required | Meaning |
| --- | --- | --- | --- |
| `id` | string (GUID) | yes | stable rule ID |
| `type` | string | yes | suggested values: `water`, `fertilize`, `harvest`, `prune`, `inspect`, `repot` |
| `intervalDays` | integer | yes | recurrence interval in days |
| `notes` | string or null | no | extra instructions |
| `startDateUtc` | string or null | no | optional starting anchor date |
| `isActive` | boolean | yes | whether the rule is active |

Rule:

- `TaskSchedule` is derived from plants plus care events and should not be stored in the data repo.

## Entity: Observation

Observations are append-only field records tied to a plant or zone. They represent what happened at a point in time.

Stored in:

- `data/plants/{species-slug}/{plant-slug}/observations/{timestamp}--{observationId}.json`

Required fields:

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | string | observation ID |
| `capturedAtUtc` | string | when the observation happened |
| `createdAtUtc` | string | metadata |
| `updatedAtUtc` | string | metadata |
| `deletedAtUtc` | string or null | metadata |
| `version` | integer | metadata |

At least one of these parent fields must be present:

- `plantId`
- `zoneId`

Recommended fields:

| Field | Type | Meaning |
| --- | --- | --- |
| `plantId` | string or null | related plant |
| `zoneId` | string or null | related zone |
| `note` | string or null | freeform field note |
| `observationType` | string | suggested values: `general`, `health`, `growth`, `soil`, `weather`, `scan`, `maintenance` |
| `latitude` | number or null | capture location |
| `longitude` | number or null | capture location |
| `gpsAccuracyMeters` | number or null | location accuracy |
| `soilMoisture` | number or null | sensor or manual reading |
| `soilPh` | number or null | sensor or manual reading |
| `soilConductivity` | number or null | sensor reading |
| `temperatureC` | number or null | local reading |
| `humidityPercent` | number or null | local reading |
| `barcodeValue` | string or null | barcode scan result |
| `qrCodeValue` | string or null | QR scan result |
| `pictureIds` | array of strings | pictures associated with this observation |
| `sourceDeviceId` | string or null | metadata |

Rules:

- Observations should almost always be appended, not edited heavily.
- If an observation has pictures, each `pictureId` must resolve to a picture sidecar metadata record for that plant.

Example:

```json
{
  "id": "72d99fe0-a370-4f4c-a90c-4206f2c2df22",
  "plantId": "9d85d070-a627-4e95-8d3d-98e37cb3e5ef",
  "zoneId": "3f7bb3a1-7d6d-4a7d-9a54-0f2ca0ed130f",
  "capturedAtUtc": "2026-03-14T12:30:00Z",
  "note": "New growth visible after two days of rain.",
  "observationType": "growth",
  "latitude": 30.123402,
  "longitude": -97.123509,
  "gpsAccuracyMeters": 2.0,
  "soilMoisture": 71.4,
  "soilPh": 6.4,
  "soilConductivity": 1.8,
  "temperatureC": 24.3,
  "humidityPercent": 87.0,
  "barcodeValue": null,
  "qrCodeValue": "PP:PLANT:9d85d070-a627-4e95-8d3d-98e37cb3e5ef",
  "pictureIds": ["a9cf3d78-1c5d-4976-9b0d-ef60771edca7"],
  "createdAtUtc": "2026-03-14T12:30:05Z",
  "updatedAtUtc": "2026-03-14T12:30:05Z",
  "deletedAtUtc": null,
  "version": 1,
  "sourceDeviceId": "android-field-phone"
}
```

## Entity: Picture

Pictures are metadata records for image files stored under each plant's `pictures/` folder.

Stored in:

- image file:
  - `data/plants/{species-slug}/{plant-slug}/pictures/{timestamp}--{pictureId}.jpg`
- sidecar metadata:
  - `data/plants/{species-slug}/{plant-slug}/pictures/{timestamp}--{pictureId}.picture.json`

Required fields:

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | string | picture ID |
| `fileName` | string | file name only, like `leaf-front.jpg` |
| `relativePath` | string | repo-relative path under `data/plants/.../pictures/` |
| `takenAtUtc` | string | timestamp for the photo |
| `createdAtUtc` | string | metadata |
| `updatedAtUtc` | string | metadata |
| `deletedAtUtc` | string or null | metadata |
| `version` | integer | metadata |

At least one parent reference should be present:

- `plantId`
- `observationId`

Optional fields:

| Field | Type | Meaning |
| --- | --- | --- |
| `plantId` | string or null | parent plant |
| `observationId` | string or null | parent observation |
| `caption` | string or null | user caption |
| `mimeType` | string or null | example: `image/jpeg` |
| `sha1Hash` | string or null | content hash |
| `width` | integer or null | image width |
| `height` | integer or null | image height |
| `sourceDeviceId` | string or null | metadata |

Rules:

- `relativePath` must stay inside the repo.
- Do not store full file system paths.
- Do not store base64 image blobs in normal operation.
- zone-level images are allowed only if we later add a zone picture folder convention; plant pictures are the default and preferred shape.

Example:

```json
{
  "id": "a9cf3d78-1c5d-4976-9b0d-ef60771edca7",
  "plantId": "9d85d070-a627-4e95-8d3d-98e37cb3e5ef",
  "observationId": "72d99fe0-a370-4f4c-a90c-4206f2c2df22",
  "fileName": "a9cf3d78-1c5d-4976-9b0d-ef60771edca7.jpg",
  "relativePath": "data/plants/althaea-officinalis/marshmallow-patch--9d85d070/pictures/2026-03-14T122958Z--a9cf3d78-1c5d-4976-9b0d-ef60771edca7.jpg",
  "takenAtUtc": "2026-03-14T12:29:58Z",
  "caption": "New top growth after rain.",
  "mimeType": "image/jpeg",
  "sha1Hash": "7f7c97db6e8c6c6cb3cce7b7bc1e8d6aa4db4515",
  "width": 3024,
  "height": 4032,
  "createdAtUtc": "2026-03-14T12:30:05Z",
  "updatedAtUtc": "2026-03-14T12:30:05Z",
  "deletedAtUtc": null,
  "version": 1,
  "sourceDeviceId": "android-field-phone"
}
```

## Relationship Rules

- A `Plant` may belong to zero or one `Zone`.
- A `Zone` may contain many `Plant` records.
- A `Plant` may have many `Observation` records.
- A `Zone` may have many `Observation` records.
- A `Picture` should always be linked to a plant and may also link to an observation.
- Do not create circular embedded objects.

Preferred reference direction:

- `Plant` points to `Zone` by `zoneId`
- `Observation` points to `Plant` and/or `Zone`
- `Picture` points to `Plant` and optionally `Observation`

## What Should Not Be Persisted

These are runtime or derived concepts and should not be stored in the repo as primary records:

- `TaskSchedule`
- "tasks due today"
- transient sync queue state
- temporary upload state
- local filesystem paths outside the repo
- full nested copies of related entities

## Legacy Compatibility Notes

The current backend codebase uses a simpler model:

- `Plant` currently contains a full `Zone` object
- `Plant` currently uses booleans like `IsIndoor` and `IsPerennial`
- `Picture` currently has an inline `Base64Data` option

The schema in this document is the cleaner target model.

Compatibility guidance:

- map legacy nested `Zone` objects to `zoneId`
- map `IsIndoor` / `IsOutdoor` to `environment`
- map `IsAnnual` / `IsPerennial` to `lifeCycle`
- treat inline `Base64Data` as migration-only, not standard storage

## AI Working Rules

If an AI agent modifies this repo, it should follow these rules:

1. Read `schema.md` before editing data files.
2. Keep JSON valid and pretty-printed.
3. Preserve unknown fields unless intentionally migrating schema.
4. Do not remove soft-deleted records unless explicitly asked to compact/archive data.
5. Do not invent new top-level files without updating `schema.md`.
6. Prefer appending observations over overwriting historical detail.
7. Keep plant pictures under each plant folder and store picture metadata in sidecar `.picture.json` files.

## Recommendation On JSON Schema

Do not start with one monolithic `schema.json`.

Better path:

1. Treat this `schema.md` as the canonical design document.
2. Let the Android app and backend align to this shape.
3. Once the shape stabilizes, add machine-readable schemas like:
   - `schemas/plant.schema.json`
   - `schemas/zone.schema.json`
   - `schemas/observation.schema.json`
   - `schemas/picture.schema.json`

That gives us:

- a document humans and AIs can reason about
- a future path to strict validation
- less churn while the model is still evolving
