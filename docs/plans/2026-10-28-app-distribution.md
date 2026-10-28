# App distribution (public releases repo + Update banner) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Every native build of Selfwise, Sagax and Sapor lands in the public repo `selfwise-labs/releases` (APK + `latest/<app>.json`), and each app shows an Update banner when that file names a newer native build than the one installed.

**Architecture:** The three private app repos' `release.yml` gain two steps (after the Android build, after the iOS build) that push to the public repo with a PAT. Each app gains one small feature folder `src/features/updates/` (pure comparison, a fetch-and-compare function, a hook with a 6-hour throttle, a banner) wired into the main screen. Spec: `docs/2026-10-28-app-distribution-design.md` in this repo.

**Tech Stack:** Expo SDK 57 / React Native 0.86, TypeScript, Jest (`jest-expo` preset), GitHub Actions, `gh` CLI, `jq`.

## Global Constraints

- The three app repos are private; `selfwise-labs/releases` is public. Nothing secret goes into the public repo or the JSON files.
- Build numbers come from EAS (`appVersionSource: remote`, `autoIncrement`). Never set them by hand.
- `latest/<app>.json` shape (from the spec):
  `{ "android": { "build": <int>, "release": "<name>", "fingerprint": "<hash>|null", "url": "<apk url>" }, "ios": { "build": <int>, "release": "<name>", "fingerprint": "<hash>|null", "url": "<testflight url>" } }`.
  The app reads only `<platform>.build` and `<platform>.url`.
- Per app constants:

  | App | Repo | App folder | TestFlight URL | APK URL |
  | --- | --- | --- | --- | --- |
  | Sapor | `selfwise-labs/sapor` | `sapor-app/` | `https://testflight.apple.com/join/mnCAbn61` | `https://github.com/selfwise-labs/releases/releases/download/sapor-latest/sapor.apk` |
  | Sagax | `selfwise-labs/sagax` | `sagax-app/` | `https://testflight.apple.com/join/EQMKsmwa` | `https://github.com/selfwise-labs/releases/releases/download/sagax-latest/sagax.apk` |
  | Selfwise | `selfwise-labs/selfwise` | `selfwise-app/` | `https://testflight.apple.com/join/hzXY9tcd` | `https://github.com/selfwise-labs/releases/releases/download/selfwise-latest/selfwise.apk` |

- The checker fetches `https://raw.githubusercontent.com/selfwise-labs/releases/main/latest/<app>.json`.
- `Constants.nativeBuildVersion` (expo-constants, present in SDK 57, deprecated but kept) is the installed native build number; no `expo-application` (it would change the native fingerprint and force a native build before the checker reaches anyone).
- Non-ASCII text goes in with the Edit/Write tools, never through a shell heredoc or a Python script (Windows Bash tool corrupts escapes and encodings). The banner copy below is ASCII only.
- Commit messages end with the session's attribution lines, exactly:

  ```
  Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>
  Claude-Session: https://claude.ai/code/session_01JvRgBstZnu4BJJYoFY72Ci
  ```

- Branch per app repo: `feat/update-banner`, from an up-to-date `main`. One PR per repo. Stage files by explicit path.
- Each app repo's checks before commit: `npx tsc --noEmit`, `npm test`, `npm run lint` from the app folder; all three must pass.

---

## Common files (identical in the three apps except the `APP` constant)

The three tasks below create the same files. The content is given here once; each task says which `APP` value to use. Replace `__APP__` with `sapor`, `sagax` or `selfwise` and `__TESTFLIGHT__`/`__APK__` with the row above.

### `src/features/updates/model.ts`

```ts
import { Platform } from 'react-native';

// What latest/<app>.json says about one platform, trimmed to what the banner needs.
export interface UpdateInfo { build: number; url: string }

export type PlatformKey = 'android' | 'ios';

export function platformKey(os: string = Platform.OS): PlatformKey | null {
  return os === 'android' || os === 'ios' ? os : null;
}

// The installed native build number as an integer, or null when unknown (Expo Go, dev builds,
// a version string that is not a number).
export function parseBuild(value: string | number | null | undefined): number | null {
  if (value === null || value === undefined) return null;
  const n = typeof value === 'number' ? value : Number.parseInt(value, 10);
  return Number.isInteger(n) && n >= 0 ? n : null;
}

// The latest file, parsed as JSON (any shape), against the installed build. Returns the newer
// build and its link, or null for "nothing to do": same or older build, missing or malformed
// section, unknown installed build.
export function newerBuild(file: unknown, platform: PlatformKey, installedBuild: number | null): UpdateInfo | null {
  if (installedBuild === null) return null;
  if (typeof file !== 'object' || file === null) return null;
  const section = (file as Record<string, unknown>)[platform];
  if (typeof section !== 'object' || section === null) return null;
  const { build, url } = section as Record<string, unknown>;
  const latest = parseBuild(typeof build === 'number' || typeof build === 'string' ? build : null);
  if (latest === null || typeof url !== 'string' || !/^https:\/\//.test(url)) return null;
  return latest > installedBuild ? { build: latest, url } : null;
}
```

### `src/features/updates/model.test.ts`

```ts
import { newerBuild, parseBuild, platformKey } from './model';

const file = {
  android: { build: 7, release: 'v10', fingerprint: null, url: 'https://github.com/selfwise-labs/releases/releases/download/__APP__-latest/__APP__.apk' },
  ios: { build: 2, release: 'v10', fingerprint: null, url: '__TESTFLIGHT__' },
};

describe('parseBuild', () => {
  it('reads integers from strings and numbers', () => {
    expect(parseBuild('6')).toBe(6);
    expect(parseBuild(6)).toBe(6);
  });
  it('is null for unknown or non-numeric versions', () => {
    expect(parseBuild(null)).toBeNull();
    expect(parseBuild(undefined)).toBeNull();
    expect(parseBuild('1.0.0-dev')).toBeNull();
    expect(parseBuild('')).toBeNull();
    expect(parseBuild(-1)).toBeNull();
  });
});

describe('platformKey', () => {
  it('accepts android and ios only', () => {
    expect(platformKey('android')).toBe('android');
    expect(platformKey('ios')).toBe('ios');
    expect(platformKey('web')).toBeNull();
  });
});

describe('newerBuild', () => {
  it('returns the newer build and its link', () => {
    expect(newerBuild(file, 'android', 6)).toEqual({ build: 7, url: file.android.url });
    expect(newerBuild(file, 'ios', 1)).toEqual({ build: 2, url: file.ios.url });
  });
  it('is null when the installed build is the same or newer', () => {
    expect(newerBuild(file, 'android', 7)).toBeNull();
    expect(newerBuild(file, 'android', 8)).toBeNull();
  });
  it('is null when the installed build is unknown', () => {
    expect(newerBuild(file, 'android', null)).toBeNull();
  });
  it('is null for a missing or malformed section', () => {
    expect(newerBuild({ ios: file.ios }, 'android', 1)).toBeNull();
    expect(newerBuild({ android: { build: 'x', url: file.android.url } }, 'android', 1)).toBeNull();
    expect(newerBuild({ android: { build: 9, url: 'ftp://nope' } }, 'android', 1)).toBeNull();
    expect(newerBuild(null, 'android', 1)).toBeNull();
    expect(newerBuild('{}', 'android', 1)).toBeNull();
  });
});
```

### `src/features/updates/check.ts`

```ts
import Constants from 'expo-constants';
import { newerBuild, parseBuild, platformKey, UpdateInfo } from './model';

export const LATEST_URL = 'https://raw.githubusercontent.com/selfwise-labs/releases/main/latest/__APP__.json';

export interface CheckDeps {
  fetch: typeof fetch;
  platform: string;            // Platform.OS
  installedBuild: string | number | null | undefined; // Constants.nativeBuildVersion
  isDev: boolean;              // __DEV__: Expo Go and dev builds never check
  url?: string;
}

// Fetches latest/<app>.json and compares it with the installed build. Never throws: every
// failure (offline, 404, non-JSON, unknown platform) is "no update".
export async function checkForUpdate(deps: CheckDeps): Promise<UpdateInfo | null> {
  if (deps.isDev) return null;
  const platform = platformKey(deps.platform);
  const installed = parseBuild(deps.installedBuild);
  if (platform === null || installed === null) return null;
  try {
    const res = await deps.fetch(deps.url ?? LATEST_URL, { headers: { accept: 'application/json' }, cache: 'no-store' });
    if (!res.ok) return null;
    return newerBuild(await res.json(), platform, installed);
  } catch {
    return null;
  }
}

// The app's own values; the hook calls this.
export function liveDeps(): CheckDeps {
  return {
    fetch: (input, init) => fetch(input, init),
    platform: require('react-native').Platform.OS as string,
    installedBuild: Constants.nativeBuildVersion,
    isDev: __DEV__,
  };
}
```

Note for the implementer: `require('react-native').Platform.OS` keeps `check.ts` importable in the pure test below without mocking react-native beyond what `jest-expo` already provides; `import { Platform } from 'react-native'` at the top is equally fine under `jest-expo` and is preferred if the test passes with it. Use the import, fall back to the require only if the test environment complains.

### `src/features/updates/check.test.ts`

```ts
import { checkForUpdate, LATEST_URL } from './check';

jest.mock('expo-constants', () => ({ __esModule: true, default: { nativeBuildVersion: '6' } }));

const body = { android: { build: 7, url: 'https://github.com/selfwise-labs/releases/releases/download/__APP__-latest/__APP__.apk' }, ios: { build: 2, url: '__TESTFLIGHT__' } };
const ok = (json: unknown): typeof fetch => jest.fn(async () => ({ ok: true, json: async () => json })) as unknown as typeof fetch;

describe('checkForUpdate', () => {
  it('fetches the latest file and reports a newer build', async () => {
    const f = ok(body);
    await expect(checkForUpdate({ fetch: f, platform: 'android', installedBuild: '6', isDev: false })).resolves.toEqual({ build: 7, url: body.android.url });
    expect(f).toHaveBeenCalledWith(LATEST_URL, expect.objectContaining({ cache: 'no-store' }));
  });
  it('uses the platform section', async () => {
    await expect(checkForUpdate({ fetch: ok(body), platform: 'ios', installedBuild: 1, isDev: false })).resolves.toEqual({ build: 2, url: body.ios.url });
  });
  it('is null on the same build', async () => {
    await expect(checkForUpdate({ fetch: ok(body), platform: 'android', installedBuild: 7, isDev: false })).resolves.toBeNull();
  });
  it('never checks in dev or Expo Go', async () => {
    const f = ok(body);
    await expect(checkForUpdate({ fetch: f, platform: 'android', installedBuild: '6', isDev: true })).resolves.toBeNull();
    await expect(checkForUpdate({ fetch: f, platform: 'android', installedBuild: null, isDev: false })).resolves.toBeNull();
    expect(f).not.toHaveBeenCalled();
  });
  it('is null on a failed response, a network error or non-JSON', async () => {
    const notFound = jest.fn(async () => ({ ok: false, json: async () => ({}) })) as unknown as typeof fetch;
    const offline = jest.fn(async () => { throw new Error('offline'); }) as unknown as typeof fetch;
    const html = jest.fn(async () => ({ ok: true, json: async () => { throw new SyntaxError('not json'); } })) as unknown as typeof fetch;
    for (const f of [notFound, offline, html]) {
      await expect(checkForUpdate({ fetch: f, platform: 'android', installedBuild: '6', isDev: false })).resolves.toBeNull();
    }
  });
});
```

### `src/features/updates/useUpdateCheck.ts`

```ts
import { useCallback, useEffect, useState } from 'react';
import { AppState } from 'react-native';
import { checkForUpdate, liveDeps } from './check';
import { UpdateInfo } from './model';

export const CHECK_INTERVAL_MS = 6 * 60 * 60 * 1000;

// Module state, so a screen that remounts does not re-check; a cold start does.
let lastCheckedAt = 0;
let lastResult: UpdateInfo | null = null;

// Runs the check at mount and whenever the app returns to the foreground, at most once per
// CHECK_INTERVAL_MS. `dismiss` hides the banner until a later check finds a newer build again.
export function useUpdateCheck(now: () => number = Date.now): { update: UpdateInfo | null; dismiss: () => void } {
  const [update, setUpdate] = useState<UpdateInfo | null>(lastResult);
  const [dismissedBuild, setDismissedBuild] = useState<number | null>(null);
  const run = useCallback(() => {
    const t = now();
    if (t - lastCheckedAt < CHECK_INTERVAL_MS) return;
    lastCheckedAt = t;
    checkForUpdate(liveDeps()).then(result => { lastResult = result; setUpdate(result); });
  }, [now]);
  useEffect(() => {
    run();
    const sub = AppState.addEventListener('change', state => { if (state === 'active') run(); });
    return () => sub.remove();
  }, [run]);
  const visible = update !== null && update.build !== dismissedBuild ? update : null;
  return { update: visible, dismiss: () => setDismissedBuild(update?.build ?? null) };
}
```

### `src/features/updates/components/UpdateBanner.tsx`

```tsx
import { Linking, Pressable, StyleSheet, Text, View } from 'react-native';
import { colors, fonts, fontSizes, radius, spacing } from '../../../theme';
import { UpdateInfo } from '../model';

// One line under the safe area on the main screen: the newer native build and where to get it.
// Android: the browser downloads the APK and offers to install it. iOS: the TestFlight page.
export function UpdateBanner({ update, onDismiss }: { update: UpdateInfo; onDismiss: () => void }) {
  return (
    <View style={styles.banner} accessibilityRole="alert">
      <Text style={styles.text} numberOfLines={1}>Version {update.build} is available</Text>
      <Pressable accessibilityRole="button" onPress={() => { void Linking.openURL(update.url); }} style={styles.action} hitSlop={8}>
        <Text style={styles.actionText}>Update</Text>
      </Pressable>
      <Pressable accessibilityRole="button" accessibilityLabel="Not now" onPress={onDismiss} hitSlop={12}>
        <Text style={styles.later}>Later</Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  banner: { flexDirection: 'row', alignItems: 'center', gap: spacing.stackMd, marginHorizontal: spacing.marginMobile, marginTop: spacing.stackSm, paddingVertical: spacing.stackSm, paddingHorizontal: spacing.stackMd, backgroundColor: colors.secondaryContainer, borderRadius: radius.lg },
  text: { flex: 1, color: colors.onSecondaryContainer, fontFamily: fonts.medium, fontSize: fontSizes.bodySm },
  action: { backgroundColor: colors.primary, borderRadius: radius.full, paddingVertical: 6, paddingHorizontal: spacing.stackMd },
  actionText: { color: colors.onPrimary, fontFamily: fonts.semiBold, fontSize: fontSizes.bodySm },
  later: { color: colors.onSecondaryContainer, fontFamily: fonts.medium, fontSize: fontSizes.bodySm },
});
```

Theme check for the implementer: `colors.secondaryContainer`, `colors.onSecondaryContainer`, `colors.primary`, `colors.onPrimary`, `spacing.stackSm/stackMd/marginMobile`, `radius.lg/full`, `fonts.medium/semiBold`, `fontSizes.bodySm` exist in `src/theme.ts` of all three apps (verified 2026-10-28 for colors, spacing and radius; confirm `fonts` and `fontSizes` names with `grep -n "fonts\|fontSizes" src/theme.ts` and adapt the two names if an app differs).

### Main screen wiring (`src/app/index.tsx`)

Add the two imports and render the banner as the first child of the screen's `SafeAreaView`:

```tsx
import { useUpdateCheck } from '../features/updates/useUpdateCheck';
import { UpdateBanner } from '../features/updates/components/UpdateBanner';
// inside the component, next to the other hooks:
const { update, dismiss } = useUpdateCheck();
// first child of <SafeAreaView ...>:
{update && <UpdateBanner update={update} onDismiss={dismiss} />}
```

### `release.yml` additions

Job-level env (add under `jobs.release`, next to `runs-on`):

```yaml
    env:
      RELEASES_TOKEN: ${{ secrets.RELEASES_TOKEN }}
```

Step after `Attach APK to GitHub Release` (replace `__APP__`, `__App__` = capitalised name, `__TESTFLIGHT__`):

```yaml
      # The public landing page and download links: selfwise-labs/releases (its
      # docs/2026-10-28-app-distribution-design.md). Skipped without the org secret RELEASES_TOKEN.
      - name: Publish the APK to selfwise-labs/releases
        if: steps.existing.outputs.count == '0' && env.RELEASES_TOKEN != ''
        run: |
          NAME="${GITHUB_REF_NAME#release/}"
          BUILD=$(jq -r '.[0].appBuildVersion' build.json)
          test -n "$BUILD" && test "$BUILD" != "null"
          gh release create "__APP__-$NAME" __APP__.apk -R selfwise-labs/releases --title "__App__ $NAME" --notes "__App__ Android build $BUILD, fingerprint ${{ steps.fp.outputs.hash }}. iPhone: __TESTFLIGHT__"
          gh release view __APP__-latest -R selfwise-labs/releases >/dev/null 2>&1 || gh release create __APP__-latest -R selfwise-labs/releases --title "__App__, latest Android build" --notes "Always the newest APK of __App__; the version is in latest/__APP__.json."
          gh release upload __APP__-latest __APP__.apk -R selfwise-labs/releases --clobber
          FILE=latest/__APP__.json
          CUR=$(gh api "repos/selfwise-labs/releases/contents/$FILE")
          NEW=$(jq -r .content <<<"$CUR" | base64 -d | jq --argjson b "$BUILD" --arg r "$NAME" --arg f "${{ steps.fp.outputs.hash }}" \
            '.android = { build: $b, release: $r, fingerprint: $f, url: "https://github.com/selfwise-labs/releases/releases/download/__APP__-latest/__APP__.apk" }')
          gh api -X PUT "repos/selfwise-labs/releases/contents/$FILE" -f message="__APP__ $NAME: Android build $BUILD" -f sha="$(jq -r .sha <<<"$CUR")" -f content="$(printf '%s\n' "$NEW" | base64 -w0)" >/dev/null
        env:
          GH_TOKEN: ${{ env.RELEASES_TOKEN }}
```

Step after `Build on EAS and submit to TestFlight`:

```yaml
      - name: Record the iOS build in selfwise-labs/releases
        if: steps.ios.outputs.enabled == 'true' && steps.existingios.outputs.count == '0' && env.RELEASES_TOKEN != ''
        run: |
          NAME="${GITHUB_REF_NAME#release/}"
          BUILD=$(npx eas-cli@latest build:list --platform ios --status finished --build-profile production --fingerprint-hash "${{ steps.fpios.outputs.hash }}" --limit 1 --json --non-interactive | jq -r '.[0].appBuildVersion')
          test -n "$BUILD" && test "$BUILD" != "null"
          FILE=latest/__APP__.json
          CUR=$(gh api "repos/selfwise-labs/releases/contents/$FILE")
          NEW=$(jq -r .content <<<"$CUR" | base64 -d | jq --argjson b "$BUILD" --arg r "$NAME" --arg f "${{ steps.fpios.outputs.hash }}" \
            '.ios = { build: $b, release: $r, fingerprint: $f, url: "__TESTFLIGHT__" }')
          gh api -X PUT "repos/selfwise-labs/releases/contents/$FILE" -f message="__APP__ $NAME: iOS build $BUILD" -f sha="$(jq -r .sha <<<"$CUR")" -f content="$(printf '%s\n' "$NEW" | base64 -w0)" >/dev/null
        env:
          EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}
          GH_TOKEN: ${{ env.RELEASES_TOKEN }}
```

Workflow validation (from the repo root): `python -I -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" .github/workflows/release.yml` (or `npx --yes yaml-lint`) must print nothing; `grep -c RELEASES_TOKEN .github/workflows/release.yml` must be 5 (job env, two `if`, two `GH_TOKEN`). The `jq` program can be checked locally: `echo '{"android":{},"ios":{}}' | jq --argjson b 7 --arg r v10 --arg f abc '.android = { build: $b, release: $r, fingerprint: $f, url: "x" }'`.

### `CLAUDE.md` of the app ("Shipping a release")

Append to the paragraph:

> Every native build is also published to the public repo `selfwise-labs/releases` (the APK under releases `<app>-<name>` and `<app>-latest`, the build numbers in `latest/<app>.json`) when the organization secret `RELEASES_TOKEN` is set; the app's Update banner (`src/features/updates/`) compares that file with `Constants.nativeBuildVersion` at start and on foreground, once per 6 hours, and opens the APK or TestFlight link. Design: `selfwise-labs/releases` `docs/2026-10-28-app-distribution-design.md`.

---

### Task 1: Sapor

**Files:**
- Create: `sapor-app/src/features/updates/model.ts`, `model.test.ts`, `check.ts`, `check.test.ts`, `useUpdateCheck.ts`, `components/UpdateBanner.tsx` (Common files, `APP = sapor`, TestFlight `https://testflight.apple.com/join/mnCAbn61`)
- Modify: `sapor-app/src/app/index.tsx` (imports, the hook next to `useRefresh`, the banner as the first child of `SafeAreaView`, before `<GestureDetector>`)
- Modify: `.github/workflows/release.yml` (job env; the two steps; `__App__` = `Sapor`)
- Modify: `CLAUDE.md` ("Shipping a release")

**Interfaces:**
- Produces: nothing other tasks consume; Tasks 2 and 3 are the same change in other repos.

- [ ] **Step 1: Branch.** In `C:\Users\crist\development\cmt-agentic\workspace\sapor`: `git checkout main && git pull --ff-only && git checkout -b feat/update-banner`.
- [ ] **Step 2: Write `model.test.ts` and run it** (`cd sapor-app && npx jest src/features/updates/model.test.ts`): fails, module missing.
- [ ] **Step 3: Write `model.ts`; run the test:** passes.
- [ ] **Step 4: Write `check.test.ts`, run it:** fails. **Write `check.ts`**, run: passes.
- [ ] **Step 5: Write `useUpdateCheck.ts` and `components/UpdateBanner.tsx`.** Check the theme names first (`grep -n "export const" src/theme.ts`).
- [ ] **Step 6: Wire `index.tsx`** as in "Main screen wiring".
- [ ] **Step 7: Workflow and CLAUDE.md** as in the Common files, with `__APP__ = sapor`, `__App__ = Sapor`, `__TESTFLIGHT__ = https://testflight.apple.com/join/mnCAbn61`. Validate the YAML and the `jq` program as described.
- [ ] **Step 8: Checks:** from `sapor-app/`: `npx tsc --noEmit`, `npm test`, `npm run lint`. All pass.
- [ ] **Step 9: Commit** by explicit paths:

```bash
git add sapor-app/src/features/updates sapor-app/src/app/index.tsx .github/workflows/release.yml CLAUDE.md
git commit -F - <<'EOF'
feat: Update banner for native releases, APK published to selfwise-labs/releases

The main screen checks latest/sapor.json in the public repo at start and on
foreground (once per 6 hours) and, when it names a newer native build than
Constants.nativeBuildVersion, shows "Version N is available" with Update
(opens the APK or the TestFlight link) and Later. Expo Go and dev builds skip
the check. release.yml publishes every native build to selfwise-labs/releases
(APK under sapor-<name> and sapor-latest, build numbers in latest/sapor.json)
when the org secret RELEASES_TOKEN is set.

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01JvRgBstZnu4BJJYoFY72Ci
EOF
```

### Task 2: Sagax

Same as Task 1 in `C:\Users\crist\development\cmt-agentic\workspace\sagax`, app folder `sagax-app/`, `APP = sagax`, `__App__ = Sagax`, TestFlight `https://testflight.apple.com/join/EQMKsmwa`. Banner placement: first child of `SafeAreaView`, before the `<View style={styles.header}>` that holds the "Home" title. Commit message: the Task 1 text with `sapor` → `sagax`.

### Task 3: Selfwise

Same as Task 1 in `C:\Users\crist\development\cmt-agentic\workspace\selfwise`, app folder `selfwise-app/`, `APP = selfwise`, `__App__ = Selfwise`, TestFlight `https://testflight.apple.com/join/hzXY9tcd`. Banner placement: first child of `SafeAreaView`, before `<KeyboardAvoidingView>`. Commit message: the Task 1 text with `sapor` → `selfwise`.

---

## After the tasks (controller)

- Per repo: check the commit trailers and byte-order marks (`git log --format=%B main..HEAD | grep Co-Authored-By`; `git log --format=%s main..HEAD | LC_ALL=C grep -c $'\xef\xbb\xbf'`), the added lines' high bytes (`git diff main..HEAD | LC_ALL=C grep '^+' | od -An -tx1 | tr -d '\n' | grep -o ' [89a-f][0-9a-f]'` must be empty or valid UTF-8 sequences), push, open the PR with the body: what changed, that no native fingerprint changes (checker ships OTA), the Expo Go check Cristian does (start Metro; the banner cannot appear in Expo Go, so a one-off check is done by temporarily returning a fixed result; or he checks on the phone after the OTA release that nothing shows), and that the workflow steps are untested until the first native release after the org secret exists.
- Start Metro on each PR branch and hand Cristian the `exp://` URL.
- Cristian: create the fine-grained PAT (owner `selfwise-labs`, repository `releases`, Contents read/write) and the org secret `RELEASES_TOKEN` for the three repos.
- First release per app after the merge: OTA only (the run must skip both native builds); phones get the checker; no banner (build numbers match `latest/*.json`).
