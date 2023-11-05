# vespa-slow-feed
A curious case of a slow feeding.

The application package is super simple.
The data is also very simple, just some IDs:

```json
{"id":"id:doc:doc::463932","fields":{"id":"463932"}}
{"id":"id:doc:doc::1371272","fields":{"id":"1371272"}}
{"id":"id:doc:doc::66719","fields":{"id":"66719"}}
{"id":"id:doc:doc::55332","fields":{"id":"55332"}}
{"id":"id:doc:doc::616545","fields":{"id":"616545"}}
{"id":"id:doc:doc::335281","fields":{"id":"335281"}}
```

To reproduce:

```shell
docker run --detach \
  --rm \
  --name vespa \
  --hostname vespa-container \
  --publish 8080:8080 --publish 19071:19071 \
  vespaengine/vespa:8.252.17

vespa version
# Vespa CLI version 8.250.43 compiled with go1.21.3 on darwin/arm64

vespa deploy --wait 60 vap

vespa status --wait 60

# when ready
cat data-100k.jsonl  \
| vespa feed - \
  --connections 8 \
  --progress 1 \
  -t http://localhost:8080
```

The rate is about 300 docs/s on a Linux laptop (and about 1300 docs/s on M1 Max laptop):
```json
{
  "feeder.seconds": 8.001,
  "feeder.ok.count": 2436,
  "feeder.ok.rate": 304.448,
  "feeder.error.count": 0,
  "feeder.inflight.count": 2230,
  "http.request.count": 2436,
  "http.request.bytes": 63823,
  "http.request.MBps": 0.008,
  "http.exception.count": 0,
  "http.response.count": 2436,
  "http.response.bytes": 176366,
  "http.response.MBps": 0.022,
  "http.response.error.count": 0,
  "http.response.latency.millis.min": 550,
  "http.response.latency.millis.avg": 4160,
  "http.response.latency.millis.max": 7531,
  "http.response.code.counts": {
    "200": 2436
  }
}
```

However, if we shuffle the input the rate is >10k docs/s on both Linux and M1 Max:
```shell
cat data-100k.jsonl  \
| shuf \
| vespa feed - \
  --connections 8 \
  --progress 1 \
  -t http://localhost:8080
```

Output:

```json
{
  "feeder.seconds": 7.045,
  "feeder.ok.count": 100000,
  "feeder.ok.rate": 14194.602,
  "feeder.error.count": 0,
  "feeder.inflight.count": 0,
  "http.request.count": 100000,
  "http.request.bytes": 2620429,
  "http.request.MBps": 0.372,
  "http.exception.count": 0,
  "http.response.count": 100000,
  "http.response.bytes": 7240858,
  "http.response.MBps": 1.028,
  "http.response.error.count": 0,
  "http.response.latency.millis.min": 20,
  "http.response.latency.millis.avg": 104,
  "http.response.latency.millis.max": 318,
  "http.response.code.counts": {
    "200": 100000
  }
}
```

Any data permutation achieves >10k docs/s.
The rate is multiple times larger of some other sort of data.
What is really strange if more resources are allocated to both container and content nodes, only the feeding rate of the shuffled data increases significantly.

To create a file with a stable operation sort:
```shell
cat data-100k.jsonl  \
| shuf > data-shuffled.jsonl

cat data-shuffled.jsonl \
| vespa feed - \
  --connections 8 \
  --progress 1 \
  -t http://localhost:8080
```
