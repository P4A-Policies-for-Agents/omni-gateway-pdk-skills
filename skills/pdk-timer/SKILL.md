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
pub fn now(&self) -> bool;
```

- `next_tick`: Sleeps for the minimum distinguishable time period
- `sleep`: Sleeps for ticks where elapsed time >= provided `Duration`
- `now`: Returns the current date and time
- `next_tick` and `sleep` return `true` when time passes, `false` if policy is unapplied/edited during execution

## Configure a Timer

```rust
#[entrypoint]
async fn configure(
   launcher: Launcher,
   Configuration(bytes): Configuration,
   clock: Clock,
   client: HttpClient,
) -> Result<()> {
   let timer = clock.period(Duration::from_secs(10));

   launcher
       .launch(on_request(|rs| request_filter(rs, &timer, &config)))
       .await?;
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

Execute tasks independent of request flow using `join!` macro (add `futures` crate):

```rust
let task = my_async_task(&timer, &config);
let launched = launcher.launch(filter);
let joined = join!(launched, task);
```

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

Use `LockBuilder` to ensure only one worker completes a task in a time period:

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

async fn my_async_task(timer: &Timer, lock: &TryLock) {
   while timer.next_tick().await {
     if let Some(acquired) = lock.try_lock() {
       // Execute task — only this worker runs it
     }
   }
}
```

Set lock `expiration` longer than total time for nested async calls. Call `refresh_lock` if awaiting results within the lock to prevent expiration.

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-timer
- Examples: Spike Policy (delayed), Metrics Policy (periodic), Block Policy (sync workers)

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `bb0f3c6`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-timer.adoc`
- **Snapshot:** 2026-04-23
