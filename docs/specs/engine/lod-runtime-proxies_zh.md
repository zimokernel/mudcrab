# LOD 运行时代理

单独管理远程表示以获取静态参考
批次无法服务：启用状态追随者、窗口/发光覆盖以及
批准的动画风景。仅限视觉，除非兼容性规则规定
否则；绝不重复脚本、库存、碰撞、任务或 AI
（运行-08）。

## 状态模型

显式状态：不可用的世界、所有者已禁用、超出范围、代理
可见，完整表示处于活动状态，转换待定。过渡状态
是可操作的，从未持续存在（RUN-01）。每个组件的可见性：

```text
想要代理=运行时启用
    和 world_context_allowed
    和有效_启用（所有者）
    并且在_policy_distance内
    和 handoff_policy_allows_proxy
```

Handoff policy is not "cell unloaded": full geometry may already show
outside attached cells, and keep/copy/replacement rules can override.
Enable-parent inversion evaluates correctly; cycles and broken parents
are compile-time errors, never infinite runtime traversal (RUN-02).

## Distance classes

Near, Far, NeverFade. Grid diameter vs radius stays explicit: for odd
diameter `d`, nominal radius is `(d - 1) / 2`, with inclusion boundaries
confirmed against the engine adapter (RUN-04). Proxies bucket spatially;
state owners keep reverse dependencies to affected proxies; no per-frame
full-scene scan (RUN-05). Polling fallback, if ever needed, carries a
configurable budget and latency.

## Epochs and persistence

Fast travel, world change, load, or reinit increments a context epoch;
asset completions from an old epoch are ignored (RUN-06). Build IDs and
schema versions validate before activation; mixed generations fail with
an actionable error, never partial scenery (RUN-09). Saves hold minimum
persistent state; spatial caches rebuild after load; stale handles never
reuse without identity validation (RUN-10).

## Glow and windows

Building bodies compile separately from controllable window overlays,
both bound to the same source reference (GLOW-01). Fixed emissive color
and external region-driven emittance stay distinct through batching
(GLOW-02); flame geometry, light sprites, glow volumes, and window
surfaces get separate asset/rule choices (GLOW-03). Fake lights simulate
luminous air only, never real light sources (GLOW-05). Preview across
day/night and weather fixtures with bounded distance multipliers
(GLOW-04).

## Animation

Each binding declares one of: autonomous loop, engine-managed effect,
explicit state channel, or unsupported. Approved list only (windmill,
waterfall, flame pattern). No door pose, movement, phase, destruction,
or script-variable sync is implied by enable/disable sync.