# vespa-slow-feed
A curious case of a slow feeding.

To reproduce:

```shell
docker run --detach \
  --rm \
  --name vespa1 \
  --hostname vespa-container \
  --network host \
  vespaengine/vespa:8.252.17

vespa deploy --wait 60 vap

vespa status --wait 60

# when ready
cat data-100k.jsonl  \
| vespa feed - \
  --connections 16 \
  --progress 1 \
  -t http://localhost:8080
```

The rate is about 300 docs/s:
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

However, if we shuffle the input the rate is:
```shell
cat data-shuffled.jsonl  \
| vespa feed - \
  --connections 16 \
  --progress 1 \
  -t http://localhost:8080
```

output:

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

```shell
cat data-100k.jsonl  \
| shuf \
| vespa feed - \
  --connections 16 \
  --progress 1 \
  -t http://localhost:8080
```
