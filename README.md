https://raw.githubusercontent.com/p-stream/theintrodb-install-guide/main/README.md

# Install TheIntroDB (`theintrodb` npm package) into a movie-web / p-stream fork

Coding-agent install guide. You are working **blind** — you have no reference implementation
to copy from, and you must **inspect the target repo** to figure out what already exists
before you start. This document is fully self-contained: every file you must create is written
out in full below.

Your goal: integrate the [`theintrodb`](https://github.com/TheIntroDB/theintrodb-npm) npm
package as the **sole** skip-segment provider, and **remove** every other skip API (febbox /
fed-skips / old introdb / raw API calls). Wire in a floating **Skip segment** button,
**auto-skip**, a **Skip Segments** settings view, and a **submit timestamps** form (authored
against a per-user TIDB API key).

Docs: <https://theintrodb.github.io/theintrodb-npm/getting-started.html>

---

## 0. First: inspect the repo and classify it

Do **not** assume anything. Look at the code and figure out which of the three states you are
in. Then follow the matching subsection of **Section 2** for your setup.

**How to classify — run these checks:**

1. Is there a `src/components/player/hooks/useSkipTime.ts` (or `useSkipSegments.ts`)?
   - **No** skip hook at all → you are **State A** ("no skip").
   - A hook exists → is it intro-only (returns a single `number`, fetches something like
     `fed-skips...` or reads an `introSkipTime` field, maybe behind a `febboxKey`/
     `ALLOW_FEBBOX_KEY` gate)? → **State B** ("legacy skip").
   - A hook exists with TIDB segment UI (a `SkipSegmentsView`, `SkipSegmentButton`,
     `AutoSkipSegments`, `TIDBSubmissionForm`, a `skipSegments` store slice)? → **State C**
     ("old TIDB").
2. Is there a `src/utils/tidb.ts`? Does it call a raw URL like
   `api.theintrodb.org/v1` or `/v2` directly?
3. Is there a `SkipSegmentButton` / `SkipSegmentsView` / `AutoSkipSegments` /
   `TIDBSubmissionForm` / `TIDBSubmissionSuccessPopout` anywhere?
4. Does `package.json` already depend on `theintrodb`?

Map your findings:

| You found… | State | What to do |
|-----------|-------|-----------|
| No `useSkipTime`, no segment UI, no `tidb.ts` | **State A** — no skip | Full install, empty repo (Section 2A) |
| `useSkipTime` returns a single intro number from a third-party skip API (fed-skips / febbox / old `introdb.app`); maybe a single `SkipIntroButton`; no TIDB UI | **State B** — legacy skip | Remove the old provider, then full install (Section 2B) |
| A `useSkipTime` that builds TIDB segments, plus `SkipSegmentsView`/`SkipSegmentButton`/`AutoSkipSegments`/`TIDBSubmissionForm`/`skipSegments` slice — but it hits the **raw API** and/or falls back to other providers | **State C** — old TIDB | Refactor: swap raw API + fallbacks for the package (Section 2C) |

If you see something in between or ambiguous, prefer to treat it as the **more
already-wired** option (State B over A, State C over B) and follow that section plus the
"remove old provider" steps — the shared components are idempotent.

---

## 1. Shared TIDB helper module (same for every state)

Create `src/utils/tidb.ts`. This centralizes the package, the params builder, the raw→internal
conversion, and submission. Keep whatever internal segment shape the rest of the player
expects — see the two variants below.

### 1.1 Package API (recap)

Install first:

```bash
pnpm add theintrodb
```

```ts
import {
  createIntroDbClient,
  getMedia,               // standalone
  submitMediaTimestamp,   // standalone
  type TIDBClient,
  type GetMediaParams,    // type only
  type NormalizedSegmentTimestamp, // type only
  type SubmitMediaTimestampInput,  // type only
} from "theintrodb";
```

Client methods (times are **milliseconds** except where noted):
- `client.getMedia(params, opts?)` → `Promise<MediaRecord>`
- `client.submitMediaTimestamp(input, opts?)` → `Promise<SubmissionResponse>`

**`getMedia` params (`GetMediaParams`):**

```ts
{ tmdbId?: number, imdbId?: string, season?: number, episode?: number, durationMs?: number }
```

`durationMs` is optional but **highly recommended** (TIDB uses it to pick the right release
cut).

**`getMedia` result (`MediaRecord`)** — every array is always present (`[]` when empty):

```ts
{
  tmdbId: number,
  type: "movie" | "tv",
  season?: number,
  episode?: number,
  intro:   NormalizedSegmentTimestamp[],
  recap:   NormalizedSegmentTimestamp[],
  credits: NormalizedSegmentTimestamp[],
  preview: NormalizedSegmentTimestamp[],
}
```

**`NormalizedSegmentTimestamp`:**

```ts
{
  startMs: number,        // 0 when the raw API said "starts at beginning"
  endMs: number | null,   // null = runs to end of media
  durationMs: number | null,
  startsAtBeginning: boolean,
  endsAtMediaEnd: boolean,
}
```

**`submitMediaTimestamp` input (`SubmitMediaTimestampInput`)** — use ONE time pair (seconds OR
ms), never both:

```ts
{
  tmdbId: number,
  type: "movie" | "tv",
  segment: "intro" | "recap" | "credits" | "preview",
  season?: number,   // required for tv
  episode?: number,  // required for tv
  videoDurationMs?: number,
  startSec?: number,          // OR startMs
  endSec?: number | null,     // OR endMs
}
```

Rules:
- `intro`/`recap`: leave `start` at `0`/`undefined` to mean "starts at beginning".
- `credits`/`preview`: leave `end` as `null`/`undefined` to mean "runs to end of media".
- Submissions **require** a per-user API key → pass `{ apiKey }` as the second argument.

> ⚠️ **Naming note.** The package uses **camelCase** (`startMs`/`endMs`) and does **not**
> return `confidence`/`submission_count`. Older in-repo code used a **snake_case** shape
> `{ start_ms, end_ms, confidence, submission_count }`. Handle the conversion once, here, so
> the rest of the player keeps whatever field names it already expects.

### 1.2 References you need from the host app

Confirm these exist in the target before wiring (they are present in every movie-web / p-stream
fork):

- `PlayerMeta` (read via the player store `usePlayerStore((s) => s.meta)`):
  ```ts
  interface PlayerMeta {
    type: "movie" | "show";
    title: string;
    tmdbId: string;   // string; convert to Number() for TIDB
    imdbId?: string;
    season?: { number: number; /* tmdbId, title */ };
    episode?: { number: number; /* tmdbId, title */ };
  }
  ```
- Current time (seconds): `usePlayerStore((s) => s.progress.time)`
- Duration (seconds): `usePlayerStore((s) => s.progress.duration)`
- Seek (seconds): `const display = usePlayerStore((s) => s.display); display.setTime(t);`

If your repo names these differently, map them and adapt the code below.

### 1.3 The module (snake_case variant — drop-in for existing p-stream-style UI)

Use this if any existing component reads `.start_ms` / `.end_ms` / `.confidence` /
`.submission_count` (typical for State B/C). It keeps the UI compiling unchanged.

```ts
import {
  createIntroDbClient,
  type GetMediaParams,
  type NormalizedSegmentTimestamp,
  type SubmitMediaTimestampInput,
} from "theintrodb";
import type { PlayerMeta } from "@/stores/player/slices/source";

const client = createIntroDbClient({
  logger: console, // optional; omit for a quiet client
});

export type TidbSegmentType = "intro" | "recap" | "credits" | "preview";

/**
 * Internal segment shape used across the player.
 * snake_case so existing UI reading `.start_ms`/`.end_ms`/`.confidence`/
 * `.submission_count` compiles unchanged. end_ms === null means "runs to media end".
 */
export interface SkipSegment {
  type: TidbSegmentType;
  start_ms: number; // 0 = start of media
  end_ms: number | null; // null = end of media
  confidence: number | null;
  submission_count: number;
}

export interface TidbResult {
  segments: SkipSegment[];
  found: boolean; // false when TIDB has no record for this media (404)
}

function toSegments(
  type: TidbSegmentType,
  timestamps: NormalizedSegmentTimestamp[],
): SkipSegment[] {
  return timestamps.map((ts) => ({
    type,
    start_ms: ts.startMs,
    end_ms: ts.endMs,
    confidence: null, // package does not expose per-segment confidence
    submission_count: 0,
  }));
}

/** Build the TIDB query params for the current player meta. */
export function metaToTidbParams(
  meta: PlayerMeta,
  durationMs?: number,
): GetMediaParams | null {
  if (meta.type === "show") {
    if (meta.season == null || meta.episode == null) return null;
    return {
      tmdbId: Number(meta.tmdbId),
      season: meta.season.number,
      episode: meta.episode.number,
      ...(durationMs ? { durationMs } : {}),
    };
  }
  return {
    tmdbId: Number(meta.tmdbId),
    ...(durationMs ? { durationMs } : {}),
  };
}

/** Fetch skip segments for the current media. THE ONLY skip provider. */
export async function fetchTidbSegments(params: GetMediaParams): Promise<TidbResult> {
  try {
    const media = await client.getMedia({ ...params });
    return {
      found: true,
      segments: [
        ...toSegments("intro", media.intro),
        ...toSegments("recap", media.recap),
        ...toSegments("credits", media.credits),
        ...toSegments("preview", media.preview),
      ],
    };
  } catch (err) {
    const status = (err as { statusCode?: number })?.statusCode;
    if (status === 404) return { found: false, segments: [] };
    // eslint-disable-next-line no-console
    console.error("TIDB getMedia failed:", err);
    return { found: false, segments: [] };
  }
}

/** Submit a single timestamp. Requires the current user's API key. */
export async function submitTidbTimestamp(
  input: SubmitMediaTimestampInput,
  apiKey: string,
): Promise<void> {
  await client.submitMediaTimestamp(input, { apiKey });
}

// Convenience alias for repos whose submit form already imports `submitIntro`.
export const submitIntro = submitTidbTimestamp;
```

### 1.4 camelCase variant (preferred for a clean green-field install)

If **nothing** in the repo reads snake_case segment fields (typical of State A), use camelCase
instead — cleaner and matches the package:

```ts
import {
  createIntroDbClient,
  type GetMediaParams,
  type NormalizedSegmentTimestamp,
  type SubmitMediaTimestampInput,
} from "theintrodb";
import type { PlayerMeta } from "@/stores/player/slices/source";

const client = createIntroDbClient({ logger: console });

export type TidbSegmentType = "intro" | "recap" | "credits" | "preview";

export interface SkipSegment {
  type: TidbSegmentType;
  startMs: number; // 0 = start of media
  endMs: number | null; // null = end of media
}

export interface TidbResult {
  segments: SkipSegment[];
  found: boolean;
}

function toSegments(
  type: TidbSegmentType,
  ts: NormalizedSegmentTimestamp[],
): SkipSegment[] {
  return ts.map((t) => ({ type, startMs: t.startMs, endMs: t.endMs }));
}

export function metaToTidbParams(
  meta: PlayerMeta,
  durationMs?: number,
): GetMediaParams | null {
  if (meta.type === "show") {
    if (meta.season == null || meta.episode == null) return null;
    return {
      tmdbId: Number(meta.tmdbId),
      season: meta.season.number,
      episode: meta.episode.number,
      ...(durationMs ? { durationMs } : {}),
    };
  }
  return { tmdbId: Number(meta.tmdbId), ...(durationMs ? { durationMs } : {}) };
}

export async function fetchTidbSegments(params: GetMediaParams): Promise<TidbResult> {
  try {
    const media = await client.getMedia({ ...params });
    return {
      found: true,
      segments: [
        ...toSegments("intro", media.intro),
        ...toSegments("recap", media.recap),
        ...toSegments("credits", media.credits),
        ...toSegments("preview", media.preview),
      ],
    };
  } catch (err) {
    const status = (err as { statusCode?: number })?.statusCode;
    if (status === 404) return { found: false, segments: [] };
    // eslint-disable-next-line no-console
    console.error("TIDB getMedia failed:", err);
    return { found: false, segments: [] };
  }
}

export async function submitTidbTimestamp(
  input: SubmitMediaTimestampInput,
  apiKey: string,
): Promise<void> {
  await client.submitMediaTimestamp(input, { apiKey });
}
```

If you choose the camelCase variant, adapt the component code in Section 3 to read
`segment.startMs`/`segment.endMs` and drop the `confidence`/`submission_count` lines.

---

## 2. Classify (again) and set up your base

Based on your Section 0 classification, do the matching setup **first**. Everything in
**Section 3** (the components) applies afterwards in every state, with the field-name variant
you picked in 1.3/1.4.

### 2A. State A — no skip code at all

Nothing to remove. Confirm `theintrodb` is installed (Section 1.1). Go to Section 3.

### 2B. State B — a legacy skip provider exists

Your `useSkipTime` (or similar) returns a single number from a third-party skip API. Remove it:

1. Locate the legacy hook (e.g. `src/components/player/hooks/useSkipTime.ts`) and the legacy
   button (e.g. `src/components/player/atoms/SkipIntroButton.tsx`).
2. Find every usage of the hook / button (grep for the function name and the button component
   name).
3. Delete the legacy button component. Delete or rewrite the legacy hook (if it's
   intro-only and you're replacing it, delete it and use the new `useSkipTime` from Section 3
   below; if other code imports it, keep the filename and replace the body).
4. In the player container file (typically `src/pages/parts/player/PlayerPart.tsx`):
   - remove `const skiptime = useSkipTime();` (or whatever the legacy hook was),
   - remove the `<SkipIntroButton ... />` usage,
   - remove the now-unused imports.
5. Remove the config/preference gate that only served the legacy provider (e.g. a
   `febboxKey`-only, `ALLOW_FEBBOX_KEY`-only skip path). **Only** remove the skip usage — if
   `febboxKey` is used by streaming/providers, keep it.
6. Delete any dead helper used only by the legacy provider (e.g. a turnstile-token fetch that
   only fed the skip call). Check for other usages before deleting.

Then go to Section 3.

### 2C. State C — an old TIDB UI exists but hits the raw API / other providers

You already have segment UI (`SkipSegmentsView`, `SkipSegmentButton`, `AutoSkipSegments`,
`TIDBSubmissionForm`, a `skipSegments` store slice). You are replacing the **data layer**
with the package and stripping fallbacks. Do this:

1. **Keep** the existing UI components and the `skipSegments` store slice. They read
   `skipSegmentsCacheKey` / `skipSegments` / `setSkipSegments` and a `SegmentData`-shaped type.
   Match your `SkipSegment` type to whatever that slice/UI already expects (see 1.3). If the UI
   reads camelCase `startMs`/`endMs`, use 1.4; if snake_case, use 1.3.
2. Locate `src/utils/tidb.ts` (if present) — it likely does a raw `fetch` to
   `https://api.theintrodb.org/v1/submit` (and/or your `useSkipTime` fetches `/v2/media`).
   **Replace** `utils/tidb.ts` with Section 1. Keep the exported function name that the submit
   form imports (e.g. `submitIntro`) by re-exporting the alias, or update the import.
3. **Rewrite the fetch hook** (`useSkipTime.ts`) so it calls `fetchTidbSegments` from the
   package and **deletes** every fallback. Full replacement is in Section 3.6.
4. Delete the fallback code paths and their now-dead helpers (fed-skips fetch, old
   `introdb.app` fetch, `mwFetch`/`proxiedFetch`/turnstile used only for skips). If a
   `useSkipTimeSource()`-style analytics helper exists that tracked which provider was used,
   set it to `"theintrodb"` or delete it.
5. Update `TIDBSubmissionForm` to call `submitTidbTimestamp` / `submitIntro` from the package
   with a properly-typed input (see Section 3.7).

Then go to Section 3 (the components already exist in State C — treat Section 3 as a
specification to verify against, and only implement the pieces you're missing).

---

## 3. Components (shared, write/compat all of these)

### 3.1 The skip-segments hook

`src/components/player/hooks/useSkipTime.ts` (or edit in place if State C). Keep the exported
type `SegmentData` aliased to your `SkipSegment`, keep the store contract
(`skipSegmentsCacheKey`/`skipSegments`/`setSkipSegments`) if that slice exists, and build:
movie → `skip-movie-{tmdbId}`; show → `skip-show-{tmdbId}-{season}-{episode}`.

```ts
import { useEffect } from "react";

import { usePlayerMeta } from "@/components/player/hooks/usePlayerMeta";
import { fetchTidbSegments, metaToTidbParams, type SkipSegment } from "@/utils/tidb";
import { usePlayerStore } from "@/stores/player/store";

// Public alias so existing consumers compile unchanged.
export type SegmentData = SkipSegment;

/** Cache key for skip segments – matches TIDB media identity (tmdbId + season + episode). */
export function getSkipSegmentsCacheKey(meta: {
  tmdbId?: string;
  type: string;
  season?: { number: number } | null;
  episode?: { number: number } | null;
} | null): string | null {
  if (!meta?.tmdbId) return null;
  if (meta.type === "movie") return `skip-${meta.type}-${meta.tmdbId}`;
  if (meta.type === "show" && meta.season != null && meta.episode != null) {
    return `skip-${meta.type}-${meta.tmdbId}-${meta.season.number}-${meta.episode.number}`;
  }
  return null;
}

// Prevent multiple components from triggering overlapping fetches for the same media.
let fetchingForCacheKey: string | null = null;

export function useSkipTime() {
  const { playerMeta: meta } = usePlayerMeta();
  const cacheKey = getSkipSegmentsCacheKey(meta ?? null);
  const skipSegmentsCacheKey = usePlayerStore((s) => s.skipSegmentsCacheKey);
  const skipSegments = usePlayerStore((s) => s.skipSegments);
  const setSkipSegments = usePlayerStore((s) => s.setSkipSegments);
  const durationMs = usePlayerStore((s) => Math.round(s.progress.duration * 1000));

  useEffect(() => {
    if (!cacheKey || !meta) return;
    if (usePlayerStore.getState().skipSegmentsCacheKey === cacheKey) return;
    if (fetchingForCacheKey === cacheKey) return;
    fetchingForCacheKey = cacheKey;

    const params = metaToTidbParams(meta, durationMs);
    if (!params) {
      fetchingForCacheKey = null;
      return;
    }

    fetchTidbSegments(params)
      .then((result) => {
        const currentKey = getSkipSegmentsCacheKey(
          usePlayerStore.getState().meta ?? null,
        );
        if (currentKey === cacheKey) {
          setSkipSegments(cacheKey, result.segments);
        }
      })
      .finally(() => {
        if (fetchingForCacheKey === cacheKey) fetchingForCacheKey = null;
      });
  }, [cacheKey, meta, durationMs, setSkipSegments]);

  // Only return segments when they're for the current media (avoid stale data)
  return cacheKey === skipSegmentsCacheKey ? skipSegments : [];
}
```

If the repo has **no** `skipSegments` slice (States A/B), create one —
`src/stores/player/slices/skipSegments.ts` — following the repo's existing `MakeSlice` store
pattern (look at `source`/`progress` slices, the `AllSlices` type in `slices/types.ts`, and the
`create(...)` composition in `store.ts`):

```ts
import type { SkipSegment } from "@/utils/tidb";
import type { MakeSlice } from "@/stores/player/slices/types";

export interface SkipSegmentsSlice {
  skipSegmentsCacheKey: string | null;
  skipSegments: SkipSegment[];
  setSkipSegments(cacheKey: string, segments: SkipSegment[]): void;
  clearSkipSegments(): void;
}

export const createSkipSegmentsSlice: MakeSlice<SkipSegmentsSlice> = (set) => ({
  skipSegmentsCacheKey: null,
  skipSegments: [],
  setSkipSegments(cacheKey, segments) {
    set((s) => {
      s.skipSegmentsCacheKey = cacheKey;
      s.skipSegments = segments;
    });
  },
  clearSkipSegments() {
    set((s) => {
      s.skipSegmentsCacheKey = null;
      s.skipSegments = [];
    });
  },
});
```

Register it in `slices/types.ts` (add to `AllSlices`) and `store.ts` (spread it in the
`create(...)` object). If the store's player `reset()` exists, clear segments there too
(`s.skipSegmentsCacheKey = null; s.skipSegments = [];`). The hook already guards against stale
data via the cache key, so this is belt-and-suspenders.

> In `usePlayerMeta()`, confirm the returned name is `playerMeta` (`const { playerMeta: meta }
> = usePlayerMeta()`); if it returns `meta` instead, destructure that. The store field names
> above match the p-stream-family convention; if your repo differs, rename them consistently
> across the slice, the hook, and the components.

### 3.2 Floating skip button

`src/components/player/atoms/SkipSegmentButton.tsx` (snake_case variant shown; adjust to
camelCase if you chose 1.4):

```tsx
import classNames from "classnames";
import { useCallback } from "react";

import { Icon, Icons } from "@/components/Icon";
import { SegmentData } from "@/components/player/hooks/useSkipTime";
import { Transition } from "@/components/utils/Transition";
import { usePlayerStore } from "@/stores/player/store";

const SEGMENT_LABELS: Record<string, string> = {
  intro: "Skip Intro",
  recap: "Skip Recap",
  credits: "Skip Credits",
  preview: "Skip Preview",
};

function shouldShowButton(
  currentTime: number,
  segment: SegmentData | null,
): "always" | "hover" | "none" {
  if (!segment) return "none";
  const currentMs = currentTime * 1000;
  const startMs = segment.start_ms ?? 0;
  const endMs = segment.end_ms ?? Infinity;
  if (currentMs >= startMs && currentMs <= endMs) {
    if (currentMs - startMs <= 10000) return "always"; // first 10s of the segment
    return "hover";
  }
  return "none";
}

function SkipButton(props: {
  className: string;
  onClick?: () => void;
  children: React.ReactNode;
}) {
  return (
    <button
      type="button"
      onClick={props.onClick}
      className={classNames(
        "font-bold rounded h-10 w-40 scale-95 hover:scale-100 transition-all duration-200",
        props.className,
      )}
    >
      {props.children}
    </button>
  );
}

export function SkipSegmentButton(props: {
  controlsShowing: boolean;
  segments: SegmentData[];
  inControl: boolean;
  onChangeMeta?: (meta: any) => void;
}) {
  const time = usePlayerStore((s) => s.progress.time);
  const duration = usePlayerStore((s) => s.progress.duration);
  const status = usePlayerStore((s) => s.status);
  const display = usePlayerStore((s) => s.display);

  const activeSegments = props.segments.filter(
    (segment) => shouldShowButton(time, segment) !== "none",
  );

  const handleSkip = useCallback(
    (segment: SegmentData) => {
      if (!display) return;
      const targetTime =
        segment.end_ms != null ? segment.end_ms / 1000 : duration;
      display.setTime(targetTime);
    },
    [display, duration],
  );

  if (!props.inControl) return null;
  if (status !== "playing") return null;
  if (activeSegments.length === 0) return null;

  return (
    <div className="absolute right-[calc(3rem+env(safe-area-inset-right))] bottom-0">
      {activeSegments.map((segment, index) => {
        const showingState = shouldShowButton(time, segment);
        const animation = showingState === "hover" ? "slide-up" : "fade";
        let bottom = "bottom-[calc(6rem+env(safe-area-inset-bottom))]";
        if (showingState === "always" && !props.controlsShowing) {
          bottom = "bottom-[calc(3rem+env(safe-area-inset-bottom))]";
        }
        const verticalOffset = index * 60;
        const adjustedBottom = bottom.replace(
          /bottom-\[calc\(([^)]+)\)\]/,
          `bottom-[calc($1 + ${verticalOffset}px)]`,
        );

        let show = false;
        if (showingState === "always") show = true;
        else if (showingState === "hover" && props.controlsShowing) show = true;

        return (
          <Transition
            key={segment.type}
            animation={animation}
            show={show}
            className="absolute right-0"
          >
            <div
              className={classNames([
                "absolute bottom-0 right-0 transition-[bottom] duration-200 flex items-center space-x-3",
                adjustedBottom,
              ])}
            >
              <SkipButton
                onClick={() => handleSkip(segment)}
                className="bg-buttons-primary hover:bg-buttons-primaryHover text-buttons-primaryText flex justify-center items-center"
              >
                <Icon className="text-xl mr-1" icon={Icons.SKIP_EPISODE} />
                {SEGMENT_LABELS[segment.type] ?? "Skip"}
              </SkipButton>
            </div>
          </Transition>
        );
      })}
    </div>
  );
}
```

- If the repo exposes player sub-components via `Player.*` (look for `components/player/
  Player.tsx` re-exporting `./atoms` + an `atoms/index.ts`), add
  `export * from "./SkipSegmentButton";` to `atoms/index.ts` so it resolves as
  `Player.SkipSegmentButton`.
- If `Icons.SKIP_EPISODE` doesn't exist, use `Icons.SKIP_FORWARD`.
- Determine how the repo passes "control" (mouse hover / watch-party host). The convention is
  an `inControl: boolean` prop slurred from the player page. Match the page's existing props.

### 3.3 Auto-skip

`src/components/player/internals/AutoSkipSegments.tsx` (renders `null`):

```tsx
import { useEffect, useRef } from "react";

import { useSkipTime } from "@/components/player/hooks/useSkipTime";
import { usePlayerStore } from "@/stores/player/store";
import { usePreferencesStore } from "@/stores/preferences";

export function AutoSkipSegments() {
  const enableAutoSkipSegments = usePreferencesStore(
    (s) => s.enableAutoSkipSegments,
  );
  const skipCredits = usePreferencesStore((s) => s.enableSkipCredits);
  const display = usePlayerStore((s) => s.display);
  const time = usePlayerStore((s) => s.progress.time);
  const meta = usePlayerStore((s) => s.meta);
  const segments = useSkipTime();

  const skippedRef = useRef<Set<string>>(new Set());

  // Reset skip state when media changes
  useEffect(() => {
    skippedRef.current.clear();
  }, [meta?.tmdbId, meta?.season?.number, meta?.episode?.number]);

  useEffect(() => {
    if (!enableAutoSkipSegments || !display) return;
    const currentSeconds = time;

    for (const segment of segments) {
      // For credits, only skip if enabled AND credits run to end of video (end_ms null)
      if (segment.type === "credits") {
        if (!skipCredits) continue;
        if (segment.end_ms !== null) continue;
      } else if (segment.end_ms === null) {
        continue; // intro/recap/preview need a defined end
      }

      const startSeconds = (segment.start_ms ?? 0) / 1000;
      const endSeconds = segment.end_ms ? segment.end_ms / 1000 : Infinity;
      const id = `${segment.type}-${startSeconds}-${endSeconds}`;

      if (currentSeconds >= startSeconds && currentSeconds < endSeconds) {
        if (!skippedRef.current.has(id)) {
          display.setTime(endSeconds === Infinity ? currentSeconds + 10 : endSeconds);
          skippedRef.current.add(id);
        }
      }
    }
  }, [enableAutoSkipSegments, skipCredits, display, time, segments]);

  return null;
}
```

If using camelCase segments (1.4), replace `start_ms`/`end_ms` with `startMs`/`endMs`.

### 3.4 Preferences

`src/stores/preferences/index.tsx` (uses `create(persist(immer<PreferencesStore>(...),
{ name: "__MW::preferences" }))`). Add to the interface, defaults, and setters (immer style):

```ts
// interface
tidbKey: string | null;        // or string; use null default
enableAutoSkipSegments: boolean;
setTidbKey(v: string | null): void;
setEnableAutoSkipSegments(v: boolean): void;

// defaults
tidbKey: null,
enableAutoSkipSegments: false,

// setters (immer)
setTidbKey(v) { set((s) => { s.tidbKey = v; }); },
setEnableAutoSkipSegments(v) { set((s) => { s.enableAutoSkipSegments = v; }); },
```

For `enableSkipCredits`:
- If the repo **already has** an `enableSkipCredits` boolean preference (used by
  "auto-play next episode" or a Skip Credits feature), **reuse it** — do not add a duplicate.
- If the repo has **no** `enableSkipCredits`, add it the same way
  (`enableSkipCredits: boolean`, default `true`, setter `setEnableSkipCredits`).

### 3.5 "Skip Segments" settings view

Check whether the repo's settings overlay uses an `OverlayRouter`/`OverlayPage` pattern
(State C definitely does). Register a route and add the view below.

`src/components/player/atoms/settings/SkipSegmentsView.tsx`:

```tsx
import { useState } from "react";

import { Button } from "@/components/buttons/Button";
import { useSkipTime } from "@/components/player/hooks/useSkipTime";
import { Menu } from "@/components/player/internals/ContextMenu";
import { TIDBSubmissionForm } from "@/components/player/TIDBSubmissionForm";
import { useOverlayRouter } from "@/hooks/useOverlayRouter";
import { usePlayerStore } from "@/stores/player/store";
import { usePreferencesStore } from "@/stores/preferences";

const TYPE_LABELS: Record<string, string> = {
  intro: "Intro",
  recap: "Recap",
  credits: "Credits",
  preview: "Preview",
};

export function SkipSegmentsView({ id }: { id: string }) {
  const router = useOverlayRouter(id);
  const display = usePlayerStore((s) => s.display);
  const segments = useSkipTime();
  const tidbKey = usePreferencesStore((s) => s.tidbKey);
  const [showForm, setShowForm] = useState(false);

  const handleSeek = (seconds: number) => display?.setTime(seconds);

  const format = (ms: number | null) => {
    if (ms == null) return "End";
    const total = Math.floor(ms / 1000);
    const m = Math.floor(total / 60);
    const s = total % 60;
    return `${m}:${s.toString().padStart(2, "0")}`;
  };

  return (
    <>
      <Menu.BackLink onClick={() => router.navigate("/")}>Skip Segments</Menu.BackLink>
      <Menu.Section>
        <div className="flex gap-2 mb-4">
          {tidbKey ? (
            <Button theme="purple" className="flex-1" onClick={() => setShowForm(true)}>
              Submit Segment
            </Button>
          ) : (
            <div className="flex-1 text-center text-sm text-type-secondary p-3">
              Enter your TheIntroDB API key in Connections settings to submit.
            </div>
          )}
        </div>
        <div className="space-y-2">
          {segments.length === 0 ? (
            <div className="text-center py-4 text-type-secondary">No skip segments available</div>
          ) : (
            segments.map((segment, i) => (
              <button
                type="button"
                key={`${segment.type}-${i}`}
                onClick={() => handleSeek(segment.start_ms / 1000)}
                className="w-full text-left p-3 rounded-xl bg-video-context-light bg-opacity-10 hover:bg-opacity-20"
              >
                <div className="flex items-center justify-between">
                  <span className="font-medium">
                    {TYPE_LABELS[segment.type] ?? segment.type}
                  </span>
                  <span className="text-sm text-type-secondary">
                    {format(segment.start_ms)} - {format(segment.end_ms)}
                  </span>
                </div>
              </button>
            ))
          )}
        </div>
      </Menu.Section>

      {showForm && tidbKey && (
        <TIDBSubmissionForm
          segment={{ type: "intro", start_ms: null, end_ms: null, confidence: null, submission_count: 0 }}
          onSuccess={() => setShowForm(false)}
          onCancel={() => setShowForm(false)}
        />
      )}
    </>
  );
}
```

(Adjust `segment.start_ms`/`end_ms` to `startMs`/`endMs` for the camelCase variant.)

**Register the route** in the settings overlay (e.g. `src/components/player/atoms/Settings.tsx`,
inside `<OverlayRouter id={id}>`):

```tsx
<OverlayPage
  id={id}
  path="/skipsegments"
  width={343}
  height={446}
>
  <Menu.Card>
    <SkipSegmentsView id={id} />
  </Menu.Card>
</OverlayPage>
```

(If the repo uses `CardWithScrollable` for long lists, use that instead.)

**Add a menu entry** in `src/components/player/atoms/settings/SettingsMenu.tsx`:

```tsx
<Menu.ChevronLink onClick={() => router.navigate("/skipsegments")}>
  Skip Segments
</Menu.ChevronLink>
```

### 3.6 Mount everything in the player page

In the file that renders `<Player.Container>` (typically
`src/pages/parts/player/PlayerPart.tsx`):

1. Add near the top:
   ```tsx
   const segments = useSkipTime();
   ```
2. Inside `<Player.Container>`, right after the existing `<Player.NextEpisodeButton ... />`
   (if present), add:
   ```tsx
   <SkipSegmentButton
     controlsShowing={showTargets}
     segments={segments}
     inControl={inControl}
     onChangeMeta={props.onMetaChange}
   />
   <AutoSkipSegments />
   ```
   where `showTargets` comes from `useShouldShowControls()` and `inControl` is the repo's
   control prop. If the repo exposes these as `Player.SkipSegmentButton` / via namespace,
   use that form.

### 3.7 Submission form

`src/components/player/TIDBSubmissionForm.tsx` (calls the package). Verify the exact
props/behavior of `Modal`/`useModal`, `AuthInputBox`, and `Button` in the target repo and adapt.

```tsx
import { useState } from "react";

import { Button } from "@/components/buttons/Button";
import { Modal, useModal } from "@/components/overlays/Modal";
import { AuthInputBox } from "@/components/text-inputs/AuthInputBox";
import { Heading3, Paragraph } from "@/components/utils/Text";
import { usePlayerStore } from "@/stores/player/store";
import { usePreferencesStore } from "@/stores/preferences";
import { submitTidbTimestamp } from "@/utils/tidb";

type SegmentType = "intro" | "recap" | "credits" | "preview";

function parseToSeconds(value: string): number | null {
  if (!value.trim()) return null;
  const mmss = value.match(/^(\d{1,3}):([0-5]\d)$/);
  if (mmss) return parseInt(mmss[1], 10) * 60 + parseInt(mmss[2], 10);
  const hhmmss = value.match(/^(\d{1,2}):([0-5]\d):([0-5]\d)$/);
  if (hhmmss)
    return (
      parseInt(hhmmss[1], 10) * 3600 +
      parseInt(hhmmss[2], 10) * 60 +
      parseInt(hhmmss[3], 10)
    );
  const num = parseFloat(value);
  return Number.isFinite(num) && num >= 0 ? num : NaN;
}

export function TIDBSubmissionForm(props: {
  segment: { type: SegmentType; start_ms: number | null; end_ms: number | null };
  onSuccess?: () => void;
  onCancel?: () => void;
}) {
  const meta = usePlayerStore((s) => s.meta);
  const tidbKey = usePreferencesStore((s) => s.tidbKey);
  const modal = useModal("tidb-submission");
  const [segment, setSegment] = useState<SegmentType>(props.segment.type);
  const [start, setStart] = useState("");
  const [end, setEnd] = useState("");
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const submit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!meta || !tidbKey) {
      setError("Media info or TIDB API key is missing.");
      return;
    }
    setSubmitting(true);
    setError(null);
    try {
      const startSeconds = parseToSeconds(start);
      const endSeconds = parseToSeconds(end);
      await submitTidbTimestamp(
        {
          tmdbId: Number(meta.tmdbId),
          type: meta.type === "show" ? "tv" : "movie",
          segment,
          ...(meta.type === "show" && meta.season && meta.episode
            ? { season: meta.season.number, episode: meta.episode.number }
            : {}),
          startSec: Number.isNaN(startSeconds ?? 0) ? undefined : (startSeconds ?? undefined),
          endSec: Number.isNaN(endSeconds ?? 0) ? null : (endSeconds ?? null),
        },
        tidbKey,
      );
      modal.hide();
      props.onSuccess?.();
    } catch (err) {
      setError(err instanceof Error ? err.message : "Submission failed.");
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <Modal id={modal.id}>
      <form onSubmit={submit} className="p-6 space-y-4">
        <Heading3>Submit Timestamps to TheIntroDB</Heading3>
        <Paragraph>
          Contribute intro/recap/credits/preview timestamps. Community review before publish.
        </Paragraph>

        <div className="flex flex-wrap gap-2">
          {(["intro", "recap", "credits", "preview"] as const).map((seg) => (
            <Button
              key={seg}
              theme={segment === seg ? "purple" : "secondary"}
              onClick={() => setSegment(seg)}
              type="button"
            >
              {seg.charAt(0).toUpperCase() + seg.slice(1)}
            </Button>
          ))}
        </div>

        <div className="grid grid-cols-2 gap-4">
          <label className="block text-sm">
            Start (leave empty for beginning)
            <AuthInputBox value={start} onChange={setStart} placeholder="2:30 or 150" />
          </label>
          <label className="block text-sm">
            End (leave empty for end of media)
            <AuthInputBox value={end} onChange={setEnd} placeholder="3:30 or 210" />
          </label>
        </div>

        {error && <p className="text-type-danger text-sm">{error}</p>}

        <div className="flex justify-end gap-2 pt-2">
          <Button theme="secondary" type="button" onClick={() => modal.hide()}>
            Cancel
          </Button>
          <Button theme="purple" type="submit" loading={submitting}>
            {submitting ? "Submitting..." : "Submit"}
          </Button>
        </div>
      </form>
    </Modal>
  );
}
```

In State C, if the existing form imports `submitIntro` from `@/utils/tidb`, the Section 1 alias
keeps it compiling — optionally rename to `submitTidbTimestamp` and use the package input shape
above.

### 3.8 API-key input (Settings → Connections)

Add a "TheIntroDB" input bound to `preferences.tidbKey`. If the repo has a per-user backend
settings sync (States B/C often do via a Connections/Settings page), thread `tidbKey` /
`setTidbKey` through the same plumbing used by an existing key (e.g. febbox / debrid). If not
(State A), a local preference input is sufficient — reads are public, only submissions need the
key.

Model the component on the repo's existing key editors (an `AuthInputBox` with placeholder
`theintrodb:user...`, `passwordToggleable`).

---

## 4. Verification checklist

1. `theintrodb` is a runtime dependency; build and lint pass
   (`pnpm install && pnpm run build && pnpm run lint`, or the repo's equivalents).
2. Grep the codebase — there is **no** remaining call to:
   - a raw `api.theintrodb.org` URL (v1/v2/v3),
   - a `fed-skips` / febbox skip endpoint,
   - `introdb.app` / "old introdb".
3. The skip hook fetches only through the `theintrodb` package.
4. Floating Skip button appears during each active segment and seeks to `end`.
5. Auto-skip skips intros/recaps/previews (when enabled) and credits (when enabled AND credits
   run to end of video).
6. The Skip Segments settings view lists fetched segments and offers a submit flow.
7. Submissions require the user's TIDB API key and POST through the `theintrodb` package.

## 5. End state

1. Every movie/episode fetch comes from `theintrodb`.
2. No other skip API is contacted.
3. The floating button, auto-skip, Skip Segments view, and submission form all run off TIDB
   data.
4. Submissions require a per-user key from <https://theintrodb.org/>.
