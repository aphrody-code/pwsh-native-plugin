# Bun test runner — reference

Jest-compatible API, no import needed for the globals. Fast (same order of magnitude as `bun run`), runs `.ts`/`.tsx` natively.

## File discovery

Default patterns (globbed from the CWD or configured `root`):
- `*.test.{js,jsx,ts,tsx}`
- `*.spec.{js,jsx,ts,tsx}`
- `*_test.{js,jsx,ts,tsx}`
- `*_spec.{js,jsx,ts,tsx}`
- `__tests__/**/*.{js,jsx,ts,tsx}`

Run with patterns:
```bash
bun test                     # all
bun test src/auth            # path filter
bun test src/auth/login      # narrower
bun test -t 'login.*admin'   # regex on test name
bun test -t '^auth'
```

## Basic API

```ts
import { describe, test, expect, beforeAll, beforeEach, afterAll, afterEach } from 'bun:test';
// or use globals directly, no import

describe('auth', () => {
  beforeAll(async () => { /* once */ });
  beforeEach(() => { /* each */ });

  test('logs in valid user', async () => {
    const r = await login('ada', 'pw');
    expect(r.ok).toBe(true);
    expect(r.user).toEqual({ name: 'ada', roles: ['admin'] });
  });

  test.skip('flaky', () => { /* */ });
  test.only('focus this', () => { /* */ });
  test.todo('implement later');
  test.each([[1,2,3],[2,2,4]])('%i + %i = %i', (a,b,c) => expect(a+b).toBe(c));
});
```

## Matchers (Jest-compatible)

`toBe`, `toEqual`, `toStrictEqual`, `toBeCloseTo`, `toBeGreaterThan`, `toMatch`, `toMatchObject`, `toContain`, `toThrow`, `toHaveBeenCalled`, `toHaveBeenCalledWith`, `toHaveBeenCalledTimes`, `toMatchSnapshot`, `toMatchInlineSnapshot`, `.resolves.*`, `.rejects.*`, `.not.*`.

## Mocks and spies

```ts
import { mock, spyOn } from 'bun:test';

const fn = mock(() => 42);
fn(); expect(fn).toHaveBeenCalledTimes(1);
fn.mockReturnValueOnce(99);
fn.mockResolvedValue({ ok: true });

const obj = { greet: (n: string) => `hi ${n}` };
const spy = spyOn(obj, 'greet');
obj.greet('ada'); expect(spy).toHaveBeenLastCalledWith('ada');
spy.mockRestore();
```

Module mocks:
```ts
import { mock } from 'bun:test';
mock.module('./db', () => ({
  query: mock(async () => [{ id: 1 }]),
}));
```

## Timers

```ts
import { setSystemTime } from 'bun:test';
setSystemTime(new Date('2030-01-01'));
setSystemTime();   // reset
// Bun also has mock timers; check `bun:test` namespace in current version
```

## DOM (via Happy DOM)

```bash
bun add -d @happy-dom/global-registrator
```
```ts
// preload.ts
import { GlobalRegistrator } from '@happy-dom/global-registrator';
GlobalRegistrator.register();
```
```bash
bun test --preload ./preload.ts
```

## Snapshots

```ts
expect(value).toMatchSnapshot();
expect(obj).toMatchInlineSnapshot(`{ "x": 1 }`);
```
```bash
bun test --update-snapshots   # update .snap files
```

## Coverage

```bash
bun test --coverage
bun test --coverage --coverage-reporter=text,lcov --coverage-dir=coverage
```

Output: text summary in terminal; `coverage/lcov.info` for CI (codecov, sonar).

Thresholds in `bunfig.toml`:
```toml
[test]
coverageThreshold = { lines = 80, functions = 80, statements = 80, branches = 70 }
```

## Concurrency and ordering

```bash
bun test --concurrent            # treat all tests as test.concurrent()
bun test --max-concurrency=10    # cap concurrent tests
bun test --randomize --seed=42   # reproducible random order
bun test --rerun-each=10         # flakiness detection
bun test --retry=3               # per-test retries
bun test --bail=1                # stop at first failure
```

## CI integration

```bash
bun test --reporter=junit --reporter-outfile=test-results.xml
bun test --dots                  # compact progress for CI logs
bun test --only-failures         # hide passing tests in output
```

Exit codes: 0 = all passed, 1 = failures, 2 = configuration error.

## Timeout

```ts
test('slow', async () => { /* */ }, 10_000);   // per-test override (ms)
```
```bash
bun test --timeout=10000         # global default (ms)
```
