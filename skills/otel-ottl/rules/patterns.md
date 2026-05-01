# Specialized OTTL patterns

## Normalize high-cardinality attributes

### Replace dynamic path segments

Replace numeric IDs and UUIDs in `url.path` and `http.route` with fixed placeholders.

```yaml
processors:
  transform/normalize-paths:
    error_mode: ignore
    trace_statements:
      - context: span
        statements:
          - replace_pattern(span.attributes["url.path"], "/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}", "/{uuid}") where span.attributes["url.path"] != nil
          - replace_pattern(span.attributes["url.path"], "/\\d+", "/{id}") where span.attributes["url.path"] != nil
          - replace_pattern(span.attributes["http.route"], "/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}", "/{uuid}") where span.attributes["http.route"] != nil
          - replace_pattern(span.attributes["http.route"], "/\\d+", "/{id}") where span.attributes["http.route"] != nil
```

### Mask IP addresses to subnet

```yaml
processors:
  transform/mask-ips:
    error_mode: ignore
    trace_statements:
      - context: span
        statements:
          - replace_pattern(span.attributes["client.address"], "(\\d{1,3}\\.\\d{1,3}\\.\\d{1,3})\\.\\d{1,3}", "$$1.0") where span.attributes["client.address"] != nil
    # Add log_statements with the same pattern using log.attributes to apply to logs.
```

### Limit attribute count and value length

```yaml
processors:
  transform/limit-attributes:
    error_mode: ignore
    trace_statements:
      - context: span
        statements:
          - limit(span.attributes, 64, [])
          - truncate_all(span.attributes, 256)
    # Add log_statements with the same pattern using log.attributes to apply to logs.
```

## Enrich telemetry with static attributes

Use the `resource` processor (not `transform`) for static resource attributes.

```yaml
processors:
  resource/static-env:
    attributes:
      - key: deployment.environment.name
        value: production
        action: upsert
      - key: k8s.cluster.name
        value: prod-us-west-2
        action: upsert
```

To copy a resource attribute down to the span or log level, use the transform processor:

```yaml
processors:
  transform/copy-resource:
    error_mode: ignore
    trace_statements:
      - context: span
        statements:
          - set(span.attributes["deployment.environment.name"], resource.attributes["deployment.environment.name"]) where resource.attributes["deployment.environment.name"] != nil
```
