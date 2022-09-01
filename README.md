# Entity Creator Assistant

![image](https://user-images.githubusercontent.com/26523841/188024750-94270268-c470-4028-b94c-7f787957cd9e.png)

This tool automates the creation of generic entity types in Dynatrace. It is a wrapper around the Generic Topology and Types feature.

Given a JSON input, the tool will create those entity types and push an `entity.discovered` metric.

## Usage

Create an API token with `settings.read`, `settings.write` and `metrics.ingest` permissions.

```
[{
    "name": "car",
    "attributes": [ "registration_number", "brand", "model", "colour", "tank_capacity", "hire_status" ]
}]
```

Test changes: `python3 app.py --input input.json --environment https://abc123.live.dynatrace.com --token dtc01.*** --dry-run`

Apply changes: `python3 app.py --input input.json --environment https://abc123.live.dynatrace.com --token dtc01.***`
 
In this case, your list of cars would be available to see at `https://abc123.live.dynatrace.com/ui/entity/list/entity:car`

### Expected Output

![image](https://user-images.githubusercontent.com/26523841/188024029-753a5d35-1f4a-4e42-8cde-97c441558adc.png)

```
-- Step 1: Read Input File and Build Entities --
-- Step 2: Check For Existing Entities in Dynatrace --
>>>> Here are the items we will create: ['car']
>>>> Creating 1 custom entities types
-- Step 3: Create Entity Types in Dynatrace --
-- Step 4: Create Dynatrace Configurations (dry mode is disabled) --

-- Step 4: Output --

Your entity type(s) have been created:

Get the entity type:
curl -X GET "https://abc123.live.dynatrace.com/api/v2/settings/objects?schemaIds=builtin%3Amonitoredentities.generic.type&scopes=environment&fields=objectId%2Cvalue" \
-H "accept: application/json; charset=utf-8" \
-H "Authorization: Api-Token dt0c01.****"

-------------------------------------------------
Outputting entity type: car
Try pushing some metrics (this will also create the entities):

curl -X POST "https://abc123.live.dynatrace.com/api/v2/metrics/ingest" \
-H "accept: */*" \
-H "Authorization: Api-Token dt0c01.*****" \
-H "Content-Type: text/plain; charset=utf-8" \
-d "entity.car.discovered,carid=1,registration_number=yourValue,brand=yourValue,model=yourValue,colour=yourValue,tank_capacity=yourValue,hire_status=yourValue 1"

View car entities:
https://abc123.live.dynatrace.com/ui/entity/list/entity:car
```

## Push Metrics

The script creates the framework (analogy: a Java class) but to create car entities (analogy: instances of the class), you need to push metrics. Entities will be created automatically.

The script automatically adds a dimension called `{entityType}id` in the above case it would be `carid`. Every entity needs a unique `carid`.

Pushing a custom metric which starts with `entity.car,carid=*` will automatically create a `car` with an ID of `1`.

> Important: If `attributes` are defined, they must be pushed with **every** metric. They cannot be left blank.

## Idempotency

This tool is idempotent. It is safe to run repeatedly (eg. in CI/CD pipelines).
