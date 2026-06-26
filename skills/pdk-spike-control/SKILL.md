---
name: pdk-spike-control
description: Use when implementing spike control to regulate traffic growth and prevent sudden surges using PDK's SpikeControlBuilder with token-bucket model, Clock ticker, bucket tiers (requests, period_in_millis), and is_allowed evaluation with TooManyRequests error handling.
---

# Skill: Using Spike Control Library Functions

## Topic: Implementation

This skill covers how to use PDK's Spike Control library to regulate traffic growth and prevent sudden surges from overwhelming upstream services. The library provides a token-bucket model driven by a Timer and Clock ticker.

## Import

```rust
use pdk::spike_control::{SpikeControlBuilder, SpikeControlError, SpikeControlHandler, Tier};
```

## Inject SpikeControlBuilder and Build a Handler

Inject `SpikeControlBuilder` and `Clock` in the `#[entrypoint]` configure function. From `Clock`, create a ticker, pass it to the builder, add at least one bucket with `Tier` values (`requests` and `period_in_millis`), and call `build`:

```rust
use anyhow::Result;
use pdk::hl::*;
use std::rc::Rc;
use std::time::Duration;

use pdk::spike_control::{
    SpikeControlBuilder, SpikeControlError, SpikeControlHandler, Tier,
};

#[entrypoint]
async fn configure(
    launcher: Launcher,
    clock: Clock,
    spike_control: SpikeControlBuilder,
) -> Result<()> {
    let ticker = Rc::new(clock.period(Duration::from_millis(100)));

    let handler = Rc::new(
        spike_control
            .new("example-spike-control".to_string())
            .with_ticker(ticker)
            .with_bucket(
                "default".to_string(),
                vec![Tier {
                    requests: 1,
                    period_in_millis: 1000,
                }],
            )
            .with_retry(200, 2)
            .build()
            .map_err(|e| anyhow::anyhow!("failed to build spike control example: {e}"))?,
    );

    launcher
        .launch(on_request({
            let handler = Rc::clone(&handler);
            move |state| request_filter(state, handler.as_ref())
        }))
        .await?;
    Ok(())
}
```

## Evaluate Traffic with is_allowed

Pass a reference to `SpikeControlHandler` into filters. Call `is_allowed` with the bucket name, traffic scope key, permit count, and boolean flags:

```rust
async fn request_filter(state: RequestState, handler: &SpikeControlHandler) -> Flow<()> {
    let _ = state.into_headers_state().await;

    match handler.is_allowed("default", "sample", 1, true).await {
        Ok(_) => Flow::Continue(()),
        Err(SpikeControlError::TooManyRequests(_)) => Flow::Break(Response::new(429)),
        Err(_) => Flow::Break(Response::new(503)),
    }
}
```

## Interpret Spike Control Errors

- `Err(SpikeControlError::TooManyRequests(_))` -- traffic exceeded the tier for that bucket and key
- Other error variants signal internal faults (timer or storage problems)

Log at debug or warn level for operational detail. Return a generic message to clients if you cannot expose internal details.

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-spike-control
- Example: Spike Policy Example

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-spike-control.adoc`
- **Snapshot:** 2026-05-14
