---
name: pdk-timer
description: Use when implementing delayed, periodic, or synchronous functions with PDK's Clock injectable and Timer, including next_tick/sleep methods, asynchronous task launching with join!, and synchronized workers with LockBuilder.
---

# Skill: Configuring Delayed, Periodic, and Synchronous Functions

## Topic: Implementation

This skill covers how to use the `Clock` injectable to build timers for delayed, periodic, and synchronous functions. Each policy supports only one timer.

## Timer Methods

```rust
pub async fn next_tick(&self) -> bool;
pub async fn sleep(&self, interval: Duration) -> bool;
pub fn now(&self) -> SystemTime;
```

- `next_tick`: Sleeps for the minimum distinguishable time period
- `sleep`: Sleeps whole ticks until the elapsed time is at least the given `Duration`. The tick is the
  timer period, so with a 10-second period `timer.sleep(Duration::from_secs(1))` sleeps 10 seconds.
- `now`: Returns the current time as a `std::time::SystemTime`. The docs page shows `-> bool`, which
  is wrong. The 1.11 crate and the Spike example return `SystemTime`.
- `next_tick` and `sleep` return `true` when time passes, `false` if policy is unapplied/edited during execution

## Configure a Timer

```rust
#[entrypoint]
async fn configure(
   launcher: Launcher,
   Configuration(bytes): Configuration,
   clock: Clock,
) -> Result<()> {
   let config: Config = serde_json::from_slice(&bytes)?;
   let timer = clock.period(Duration::from_secs(10));

   launcher
       .launch(on_request(|rs| request_filter(rs, &timer, &config)))
       .await?;
   Ok(())
}
```

Then use in wrapped functions:

```rust
let slept = timer.sleep(duration).await;
let tick = timer.next_tick().await;
```

### Get Current Time

Use the `.now` function to get the current date and time:

```rust
async fn filter(
    state: RequestState,
    timer: &Timer,
) {
    let state = state.into_headers_state().await;
    let now: chrono::DateTime<chrono::Utc> = timer.now().into();
    state.handler().set_header("Date", now.format("%a, %d %b %Y %H:%M:%S GMT").to_string().as_str());
}
```

## Release the Clock

Create a new timer with a different period by releasing the clock:

```rust
pub fn release(self) -> Clock;

let initial_timer = clock.period(Duration::from_millis(500));
some_initial_task(&initial_timer).await;
let other_timer = initial_timer.release().period(Duration::from_millis(3000));
some_other_task(&other_timer).await;
let clock = other_timer.release();
```

## Launch Asynchronous Tasks

Execute tasks independent of request flow using `join!` macro (add `futures` crate). Call the task
function and `launch` without awaiting either one, then await them together with `join!`:

```rust
let task = my_async_task(&timer, &config);   // not awaited
let launched = launcher.launch(filter);      // not awaited
let joined = join!(launched, task);
```

Each worker runs its own copy of the task. Put a loop inside the task for periodic work, and use a
lock (below) when only one worker should do it.

A task launched this way can make HTTP calls with an `HttpClient` injected into `configure`. The
official Block example fetches its IP list from a timer loop like this, even though the HTTP call
docs page says calls are only allowed from the request or response filter.

### Periodic Tasks

```rust
async fn my_async_task(timer: &Timer, config: &Config) {
    while timer.next_tick().await {
        // Execute periodic task
    }
}
```

### Multiple Tasks

```rust
let task1 = my_async_task1(&timer, &config);
let task2 = my_async_task2(&timer, &config);
let launched = launcher.launch(filter);
let joined = join!(launched, task1, task2);
```

## Sync Workers with Locks

Use `LockBuilder` to ensure only one worker completes a task in a time period, for example when the
task calls an HTTP service with usage limits. A policy can create several locks, each with its own
ID:

```rust
#[entrypoint]
async fn configure(
    launcher: Launcher,
    clock: Clock,
    lock: LockBuilder,
) -> Result<()> {
    let lock = lock
        .new(ID.to_string())
        .expiration(Duration::from_secs(20))
        .build();
    let task = my_async_task(&timer, &lock);
    // ...
}

async fn my_async_task(timer: &Timer, client: &HttpClient, service: &Service, lock: &TryLock) {
   while timer.next_tick().await {
     // Lock not acquired: another worker runs the task. Skip and wait for the next tick.
     let Some(acquired) = lock.try_lock() else { continue };

     let response = client.request(service).get().await;

     // After every await, confirm the lock is still held. If it expired, this worker was
     // considered unresponsive and another worker may be running the same task, so discard
     // the result.
     if !acquired.refresh_lock() {
       continue;
     }
     // ... use `response` ...
   }
}
```

Set lock `expiration` longer than the total time of the awaited calls inside the lock. Call
`refresh_lock` after each await: it extends the lock and returns `false` if another worker has
taken it.
On `false`, abort that run of the task, as the Block example does when it discards the fetched data.

## HTTP Calls After Ticks (PDK 1.10+)

PDK 1.10 fixes HTTP calls made after awaiting timer ticks so they route through the request context
instead of the root context handler. Upgrade to 1.10+ when a request or WebSocket flow awaits a
tick and then calls an HTTP service.

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-timer
- Examples: Spike Policy (delayed), Metrics Policy (periodic), Block Policy (sync workers)

## Source Ref

- **Repo:** `mulesoft/docs-gateway`
- **Branch:** `latest`
- **File:** `pdk/1.11/modules/ROOT/pages/policies-pdk-configure-features-timer.adoc`
- **Release notes:** https://docs.mulesoft.com/release-notes/pdk/pdk-release-notes (1.10.0)
- **Verified against:** `pdk-classy` 1.11.0 (`Timer::now`) and the 1.11.0 `spike` and `block`
  examples (lock refresh, HTTP calls from a timer task); where they differ from the docs page, they win
- **Snapshot:** 2026-11-24
