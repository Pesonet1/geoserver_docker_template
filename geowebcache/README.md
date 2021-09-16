# Geowebcache (standalone)

This folder contains configurations to run Geowebcache as standalone application. Ideally seperate Geowebcache instance is used to run seeding processes and other to serve seeded tiles. Configuration assumes that user wants to attach volume to running container for seeding tiles to remote location. In Azure this can be files share / blob storage container.

# Running

```
docker build . -f .\Dockerfile -t geowebcachepoc
docker run --rm -it --volume ./cache/tmp/defaultCache -p 8080:8080 geowebcachepoc
```

## Seeding

Geowebcache REST API can be used for triggering seeding processes.

Add authorization header for each request

Get GWC layers

```
GET http://localhost:8080/gwc/rest/layers/
```

Post seed request

```
POST http://localhost:8080/gwc/rest/seed/poc:kunnat_2019.json

BODY {
  "seedRequest": {
    "name": "poc:kunnat_2019",
    "srs": {
      "number": 3067
    },
    "zoomStart": 0,
    "zoomStop": 5,
    "format": "image/png",
    "type": "seed",
    "gridSetId": "JHS180",
    "threadCount": 4
  }
}
```

Get seed status

```
GET http://localhost:8080/gwc/rest/seed/poc:kunnat_2019.json

RESPONSE {
  "long-array-array": [
    [
      4592, # tiles completed
      5645, # tile estimate count
      119, # time remaining
      1, # task id
      1 # task status (-1 = ABORTED, 0 = PENDING, 1 = RUNNING, 2 = DONE)
    ]
  ]
}
```

Kill all seeding processes or only for one layer

```
POST http://localhost:8080/gwc/rest/seed.json?kill_all=all
POST http://localhost:8080/gwc/rest/seed/poc:kunnat_2019.json?kill_all=all
```
