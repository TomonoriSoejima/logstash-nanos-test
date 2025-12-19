# Logstash Nanosecond Precision Test

This test demonstrates the issue where Logstash truncates timestamps to millisecond precision, even when targeting a `date_nanos` field.

## Quick Test (Logstash only)

Test Logstash parsing without Elasticsearch:

```bash
docker run --rm -i \
  -v "$(pwd)/logstash.conf:/usr/share/logstash/pipeline/logstash.conf" \
  docker.elastic.co/logstash/logstash:9.2.0 \
  < test-logs.json
```

**Expected result:** You'll see timestamps truncated to `.123` instead of preserving `.123456789`

## Full Test (with Elasticsearch)

### Step 1: Start Elasticsearch and Kibana

```bash
docker-compose up -d elasticsearch kibana
```

Wait ~30 seconds for Elasticsearch to be ready.

### Step 2: Create index with date_nanos mapping

```bash
curl -X PUT "localhost:9200/nanos-test" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "@timestamp": {
        "type": "date"
      },
      "log_timestamp": {
        "type": "date_nanos"
      },
      "timestamp": {
        "type": "keyword"
      },
      "message": {
        "type": "text"
      }
    }
  }
}
'
```

### Step 3: Ingest data via Logstash

```bash
docker run --rm -i \
  --network logstash-nanos-test_elastic \
  -v "$(pwd)/logstash-es.conf:/usr/share/logstash/pipeline/logstash.conf" \
  docker.elastic.co/logstash/logstash:9.2.0 \
  < test-logs.json
```

### Step 4: Query and verify

```bash
curl -X GET "localhost:9200/nanos-test/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "fields": [
    {
      "field": "@timestamp",
      "format": "strict_date_optional_time_nanos"
    },
    {
      "field": "log_timestamp",
      "format": "strict_date_optional_time_nanos"
    }
  ],
  "_source": true
}
'
```

### Expected Results

- Original `timestamp` field (keyword): `2025-12-15T10:00:00.123456789Z` ✅ Full precision preserved
- `@timestamp` (date): `2025-12-15T10:00:00.123Z` ❌ Truncated to milliseconds
- `log_timestamp` (date_nanos): `2025-12-15T10:00:00.123000000Z` ❌ Truncated despite date_nanos mapping

**This proves the issue:** Logstash date filter truncates nanoseconds BEFORE indexing, so even a `date_nanos` field can't preserve them.

## Cleanup

```bash
docker-compose down -v
```

## Notes

- The Full Test uses a one-off Logstash container that reads from stdin and outputs to the running Elasticsearch instance
- This approach is more reliable than using file input, which can have issues with re-ingestion on restarts
- The `--network logstash-nanos-test_elastic` flag connects the Logstash container to the same network as Elasticsearch

## The Problem

The Logstash `date` filter uses Joda-Time internally, which only supports millisecond precision. Even though:
- The source data has nanosecond precision
- The target field is mapped as `date_nanos` in Elasticsearch
- No errors are thrown

...the precision is lost during parsing in Logstash, before the data ever reaches Elasticsearch.
