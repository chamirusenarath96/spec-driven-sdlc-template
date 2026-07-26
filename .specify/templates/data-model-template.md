# Data model: <feature name>

Only include this file if the feature touches persisted data. Delete it from
the spec directory otherwise — don't ship an empty placeholder.

## Entities

### <Entity>

| Field | Type | Required | Notes |
|---|---|---|---|
| | | | |

## Relationships

<Entity>-to-<Entity>: <cardinality and description>

## Migration

<Forward migration description. State explicitly whether it is
backwards-compatible with the currently deployed code, and whether it can be
rolled back safely.>
