+++
title = "Transparent Compression in Valkey GLIDE: Reduce Memory With a Single Line of Code"
description = "Learn how to enable automatic compression in Valkey GLIDE to reduce memory usage by up to 49% without modifying your application code."
date = 2026-01-27 01:01:01
authors = ["dknowles"]

[extra]
featured_image = "/assets/media/featured/random-03.webp"
+++

If you're caching JSON API responses, storing user sessions, or buffering HTML fragments in Valkey, there's a good chance your data is highly compressible while you're still paying full price for every byte. At scale, redundant data adds up fast. ~1KB JSON payloads across millions of keys means gigabytes of memory that could be reclaimed without losing a single field.

Transparent compression in [Valkey GLIDE](https://github.com/valkey-io/valkey-glide) offers a seamless solution to this problem. When you write data with a `SET` command, GLIDE compresses it before sending it to the server. When you read it back with `GET`, GLIDE decompresses it automatically. Your application code doesn't change — you just flip a switch in the client configuration:

```python
from glide import (
    GlideClient,
    GlideClientConfiguration,
    NodeAddress,
    CompressionConfiguration,
)

config = GlideClientConfiguration(
    addresses=[NodeAddress(host="localhost", port=6379)],
    compression=CompressionConfiguration(enabled=True)  # That's it.
)
client = await GlideClient.create(config)

# Everything else stays exactly the same
await client.set("user:1001:profile", json.dumps(user_data))
profile = json.loads(await client.get("user:1001:profile"))
```

In our benchmarks, this single configuration change delivered 28–49% memory savings depending on algorithm choice, with LZ4 showing zero measurable throughput impact and Zstandard (zstd) nearly halving memory usage at the cost of some write throughput. This post walks through how the feature works, what the benchmarks show, and how to choose the right settings for your workload.

## How It Works

When you execute a `SET` command with compression enabled, GLIDE runs through a short decision path on the client side:

1. Is the value larger than the minimum size threshold (default: 64 bytes)? If not, send it uncompressed — the 5-byte header overhead would negate any savings and small values are unlikely to be compressible.
2. Compress the value using the configured algorithm (zstd or LZ4).
3. Is the compressed result actually smaller than the original? If not, send the original instead.
4. Prepend a 5-byte header that identifies the data as GLIDE-compressed, and send it to the server.

On the read path, the process reverses. GLIDE checks for the header, decompresses if present, and returns the original value. If the header isn't there, the value is returned as-is. This allows  compression-enabled clients to seamlessly read uncompressed data written by older clients.

GLIDE uses a 5-byte header (`[Magic Prefix: 3 bytes][Version: 1 byte][Backend ID: 1 byte]`) to tag compressed values. The backend ID means a zstd-configured client can read LZ4-compressed data and vice versa. All GLIDE language bindings (Python, Node.js, Java, Go) share the same header format, so compressed data written from one language can be read from another.

A few safety-by-default choices keep compression from ever getting in the way: if compression fails for any reason, GLIDE silently falls back to uncompressed data. Data that already carries the GLIDE header won't be double-compressed. And after compressing, GLIDE compares sizes — if compression didn't help, the original goes through unchanged.

## Benchmark Results

We benchmarked compression using a parallel multi-process architecture on Amazon EC2 r7g.2xlarge instances (8 vCPUs, 64 GB RAM, AWS Graviton3) with the client and Valkey 8.0 server in the same AWS VPC. The test corpus was JSON documents averaging ~1,884 bytes per value in order to push compression to its limits. We swept a matrix of configurations across client counts, parallel processes, and pipeline batch sizes.

The compression itself happens in Rust via FFI and is identical across all GLIDE language bindings, though baseline throughput varies by language.

## Memory Savings

The headline numbers are consistent regardless of throughput or concurrency — they depend only on the data and the algorithm:

| Backend | Server Memory | Reduction |
|---------|--------------|-----------|
| No compression | 21.5 MB | — |
| zstd (level 3) | 10.9 MB | 49.2% |
| LZ4 (level 0) | 15.5 MB | 27.8% |

Zstd saves nearly twice as much memory as LZ4 on this JSON workload. But ~2KB JSON payloads are just one scenario. In practice, you're caching all kinds of data at all kinds of sizes. We benchmarked three common data types — JSON, HTML, and session data — across small, medium, and large value sizes to show what savings actually look like across real workloads.

### Memory Savings by Data Type and Value Size

| Data Type | Value Size | Avg Bytes | zstd Savings | LZ4 Savings |
|-----------|-----------|-----------|-------------|------------|
| JSON | Small | 38 | 0.0% | 0.0% |
| JSON | Medium | 98 | 0.8% | 0.0% |
| JSON | Large | 461 | 27.6% | 15.0% |
| JSON | Extra-Large | 1,884 | 49.3% | 31.8% |
| HTML | Small | 193 | 20.0% | 10.5% |
| HTML | Medium | 556 | 37.2% | 27.9% |
| HTML | Large | 1,247 | 49.0% | 38.9% |
| Session | Small | 198 | 11.8% | 0.0% |
| Session | Medium | 480 | 16.9% | 0.0% |
| Session | Large | 951 | 24.1% | 11.5% |

A few patterns jump out:

**Value size matters more than data type.** Below ~100 bytes, neither algorithm saves meaningful memory — the 5-byte header overhead and Valkey's per-key metadata dominate. Above ~500 bytes, both algorithms deliver substantial savings across all data types.

**HTML compresses well.** Repeated tags, attributes, and structural patterns give both algorithms plenty to work with. Even small HTML fragments (~193 bytes) see 10–20% savings, and large fragments hit 39–49%.

**Session data is the toughest.** Session payloads tend to be more random — UUIDs, tokens, timestamps — with less structural repetition. zstd still manages 12–24% savings, but LZ4 struggles with smaller session values (0% savings at ~200 and ~480 bytes) because the compressed output isn't smaller than the original after accounting for the header.

**zstd consistently beats LZ4 on compression ratio.** Across every data type and size, zstd saves more memory. The gap is widest on highly compressible data and narrowest on small or low-redundancy data.

These are total server memory reductions as reported by Valkey's `INFO memory` — they include per-key overhead (key storage, metadata, allocator alignment) that doesn't compress, so the effective memory ratio is lower than the raw algorithm compression ratio you'd see compressing the same data outside of Valkey.

![Memory comparison: zstd vs LZ4 vs uncompressed](graph_parallel_memory.png)
![Memory savings across data types](graph_memory_savings.png)

## The Core Tradeoff: Memory vs Throughput

The two algorithms represent a clear tradeoff:

- **LZ4** delivers 27.8% memory savings with effectively zero throughput impact. Across all configurations in our sweep, SET throughput ratio vs uncompressed ranged from 0.89x to 1.14x. Some configurations were actually *faster* with LZ4 because smaller payloads reduced network transfer time. Peak throughput with LZ4: 180K SET TPS / 245K GET TPS — identical to the uncompressed baseline.

- **Zstd** delivers 49.2% memory savings but with a real CPU cost. SET throughput ratios ranged from 0.22x to 0.89x, and the pattern is consistent: the higher the baseline throughput, the worse the ratio. Without batching, zstd costs 11–42% of SET throughput. With aggressive batching, the cost grows because compression becomes CPU-bound.

Here's a representative slice at 4 parallel processes with 10 clients each:

| Batch Size | Baseline SET | zstd SET | zstd Ratio | LZ4 SET | LZ4 Ratio |
|------------|-------------|----------|------------|---------|-----------|
| 1 | 15,060 | 9,295 | 0.62x | 14,718 | 0.98x |
| 5 | 58,401 | 22,268 | 0.38x | 56,327 | 0.98x |
| 10 | 100,666 | 28,968 | 0.29x | 92,130 | 0.95x |
| 20 | 143,629 | 34,010 | 0.24x | 128,227 | 0.90x |

It's clear that when batching pushes throughput higher, zstd's CPU cost becomes the bottleneck. LZ4 stays within a few percent of baseline across the board.

![Compression overhead heatmap across configurations](graph_parallel_ratio_heatmap.png)

One important note: batching is the real throughput lever. Going from batch size 1 to 20 takes baseline SET throughput from 15K to 144K TPS — a 9.5x improvement that dwarfs any compression effect. Even with zstd, batched operations at B=20 outperform unbatched uncompressed operations by 2.3x. Optimize your batching strategy first, then choose your compression backend.

![Batch size impact on throughput](graph_parallel_batch_impact_c10.png)

## Choosing the Right Configuration

**Start with LZ4** if throughput matters. Switch to zstd if you need maximum memory savings and have throughput headroom. The savings you'll see depend heavily on your data type and value size — HTML compresses best, session data compresses least, and anything under 100 bytes isn't worth compressing. Skip compression entirely for already-compressed data (images, video, pre-compressed content).

For value sizes:
- **Under 100 bytes**: Skip compression. Neither algorithm saves meaningful memory at this size — the 5-byte header overhead and Valkey's per-key metadata dominate (38-byte JSON = 0.0% savings, 98-byte JSON = 0.8% zstd / 0.0% LZ4).
- **100–500 bytes**: Savings vary by data type. HTML compresses well even at ~193 bytes (20% zstd / 10.5% LZ4). Session data sees 12% zstd savings at ~198 bytes. LZ4 may not help at this size for session data (0% savings). Worth enabling with zstd; test with LZ4.
- **500–1,000 bytes**: Solid savings across all data types. Expect 17–49% with zstd and 0–39% with LZ4 depending on data type. HTML compresses best; session data compresses least.
- **Over 1,000 bytes**: Strongly recommended. Expect 24–50% with zstd and 12–39% with LZ4. JSON and HTML hit 49% zstd savings at 1.2–1.9KB.

For batched operations: prefer LZ4. Zstd throughput drops significantly as batch size increases.

Here's a complete configuration example:

```python
from glide import (
    GlideClient,
    GlideClientConfiguration,
    NodeAddress,
    CompressionConfiguration,
    CompressionBackend,
)

config = GlideClientConfiguration(
    addresses=[NodeAddress(host="localhost", port=6379)],
    compression=CompressionConfiguration(
        enabled=True,
        backend=CompressionBackend.LZ4,     # or CompressionBackend.ZSTD
        compression_level=0,                 # lz4: -128 to 12; zstd: -131072 to 22
        min_compression_size=64              # Skip values smaller than 64 bytes
    )
)

client = await GlideClient.create(config)
```

Compression works identically with `GlideClusterClient` — just pass the same `CompressionConfiguration` to your cluster config.

## Gradual Rollout

One of the most practical aspects of GLIDE's compression design is that you don't need a big-bang migration. Compression-enabled clients read uncompressed data transparently. Clients configured with zstd can read LZ4-compressed data and vice versa. This means you can roll out incrementally: deploy new clients with compression enabled, and as keys expire or get updated through normal application flow, data naturally migrates to compressed format — no migration scripts, no downtime.

## Conclusion

Transparent compression gives you a meaningful reduction in memory usage with minimal effort. LZ4 is effectively free at 28% savings; zstd nearly halves memory at the cost of write throughput. The feature is available today across all GLIDE language bindings and requires only a single configuration change.

We'd love to hear about your experience with compression — what data types you're compressing, what savings you're seeing, and what would make the feature more useful. Join the conversation on [GitHub Discussions](https://github.com/valkey-io/valkey-glide/discussions).

To get started:
- [Valkey GLIDE GitHub Repository](https://github.com/valkey-io/valkey-glide)
- [Compression Benchmark Tool & Data](https://github.com/valkey-io/valkey-glide/tree/main/benchmarks/compression_benchmark)
