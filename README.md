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

The tool will print a sample curl request:

Create the entity screens automatically:
```
curl -X POST "https://abc123.live.dynatrace.com/api/v2/metrics/ingest" \
-H "accept: */*" \
-H "Authorization: Api-Token dt0c01.*****" \
-H "Content-Type: text/plain; charset=utf-8" \
-d "entity.car.discovered,carid=1,registration_number=yourValue,brand=yourValue,model=yourValue,colour=yourValue,tank_capacity=yourValue,hire_status=yourValue 1"
```

Push any metric such as the amount of fuel remaining in the tank:
```
curl -X POST "https://abc123.live.dynatrace.com/api/v2/metrics/ingest" \
-H "accept: */*" \
-H "Authorization: Api-Token dt0c01.*****" \
-H "Content-Type: text/plain; charset=utf-8" \
-d "entity.car.fuel_level,carid=1,registration_number=yourValue,brand=yourValue,model=yourValue,colour=yourValue,tank_capacity=yourValue,hire_status=yourValue 1"
```

![image](https://user-images.githubusercontent.com/26523841/188027706-f48581db-7b73-484e-8229-4613079a7460.png)

## Push Events

Use the `/api/v2/events` endpoint. You will need an API token with `events.ingest` permissions.

Target your entity using either it's unique id or a property such as `registration_number` or `brand`. Note that attributes are case sensitive `brand != Brand`.

You can push problem opening events, information events, deployment events or anything else that the `api/v2/events` endpoint supports.

### Info Event to Signal a Car Hire is Overdue

![image](https://user-images.githubusercontent.com/26523841/188032725-e2504081-3efa-4cec-8963-201afaa19ee3.png)

This is just a sample, set the properties to whatever you need.

```
curl -X POST "https://abc123.live.dynatrace.com/api/v2/events/ingest" \
-H "accept: application/json; charset=utf-8" \
-H "Authorization: Api-Token dt0c01.****" \
-H "Content-Type: application/json; charset=utf-8" \
-d "{\"eventType\":\"CUSTOM_INFO\",\"title\":\"car is overdue\",\"timeout\":1,\"entitySelector\":\"type(entity:car),registration_number(ABC123)\",\"properties\":{\"is_overdue\":\"true\",\"driver\":\"Bob Smith\",\"last_location\":\"48.060457,1.262060\"}}"
```

## Problem Event to Signal all Fords need a Recall

![image](https://user-images.githubusercontent.com/26523841/188035483-eb2b3cd4-abc2-43a3-9d4b-b590d2096f0b.png)
![image](https://user-images.githubusercontent.com/26523841/188035259-9ffe9fd8-360f-416b-b131-e41b44a5bbb2.png)

Push an event targeting all Ford vehicles in your fleet because they need to be recalled.

```
curl -X POST "https://abc123.live.dynatrace.com/api/v2/events/ingest" -H "accept: application/json; charset=utf-8" -H "Authorization: Api-Token dt0c01.****" -H "Content-Type: application/json; charset=utf-8" -d "{\"eventType\":\"CUSTOM_ALERT\",\"title\":\"Ford recall notice\",\"timeout\":1,\"entitySelector\":\"type(entity:car),brand(Ford)\",\"properties\":{\"description\":\"Recall notice #445 for all Ford vehicles\",\"recall-notice\":\"https://example.com/recall/445\",\"recall-priority\":\"immediately\"}}"
```

## Idempotency

This tool is idempotent. It is safe to run repeatedly (eg. in CI/CD pipelines).
