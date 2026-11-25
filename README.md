# Gnome Tavern

An on-chain tavern for gnomes, built as a single Gno realm for
**The Gnome Tavern Hackathon** (Gno.land, DoraHacks).

Patrons check in under a nickname, order drinks from an on-chain menu
paying in native GNOT, leave tips, and complete daily quests. Every
regular competes on the **Hall of Fame** leaderboard. The tavern keeper
(the deploying account) stocks the kegs and writes the quest board.

No custom token, no off-chain backend: the realm *is* the tavern, and
its `Render()` output — served by gnoweb — is the user interface.

**Realm path (Onyx / test17, chain-id `onyx-1`):**
`gno.land/r/g1mqx99g0wjfzmjgpespuhq26kd59f3ek5cmkntv/gnometavern`

## Features

- **Check-ins** — `CheckIn(nickname)` registers a patron (nickname is
  locked at the first visit) and counts repeat visits.
- **Drink menu & stock** — five seeded drinks with prices in `ugnot`
  and finite keg stock. `OrderDrink(id)` requires an exact-or-higher
  payment sent with the call; overpayment is recorded as a tip, the
  keg count decreases, and the till grows.
- **Tipping** — `Tip()` sends any amount straight to the till and the
  patron's record.
- **Daily quests** — a quest board (4 seeded quests). `CompleteQuest(id)`
  awards points once per patron per quest per day, using chain block
  time — come back tomorrow for another round.
- **Hall of Fame** — score = visits + 2·drinks + quest points +
  tips/100000, rendered as a top-10 leaderboard.
- **Keeper tools** — only the keeper can `AddMenuItem`, `Restock` and
  `AddQuest`.
- **gnoweb pages** — home (stats, menu, quests, leaderboard), `menu`,
  `quests`, and a personal `patron/<address>` card.

## Security model

Following the official Gno security guide for payable realm functions:

- Caller identity always comes from `cur.Previous()` — never from an
  address parameter (there are none to spoof).
- `OrderDrink`/`Tip` require a *direct user call*
  (`cur.Previous().IsUserCall()`), so other realms cannot spend a
  patron's funds through the tavern, and the payment is read from
  `unsafe.OriginSend()` — the coins actually attached to the call.
- Underpayment, unknown items, sold-out kegs, repeat daily quests and
  non-keeper admin calls all abort before any state changes.
- Text input (nicknames, menu and quest names) is length-capped and
  stripped of control characters so the rendered pages stay clean.

## Layout

```
src/gnometavern/
├── gnomod.toml            module gno.land/r/<keeper>/gnometavern
├── gnometavern.gno        the realm (state, logic, Render)
└── gnometavern_test.gno   7 unit tests
tools/
└── rpc_relay.py           local RPC relay (see "Sandbox note" below)
```

## Run the tests

```sh
# Gno toolchain v1.8.0 (https://docs.gno.land/latest/getting-started/installation)
export PATH="$HOME/.gno/bin:$PATH"
export GNOROOT=/path/to/gno-source   # a checkout of gnolang/gno at v1.8.0

cd src/gnometavern
gno lint ./...
gno test -v .
```

Latest run: **7/7 tests pass** (`TestCheckIn`, `TestOrderDrink`,
`TestTip`, `TestCompleteQuest`, `TestKeeperOnly`, `TestRender`,
`TestHallOfFameRanking`), covering payments (exact / overpay-as-tip /
underpay-abort), stock, daily-quest reset via block time, keeper-only
admin, rendering, and leaderboard ordering.

Two testing quirks worth knowing (documented in the test file):
`testing.SetRealm`/`SetOriginSend` must be called inline in the test
body, send before realm; and realm panics are asserted with the
`revive()` builtin because a plain `recover()` cannot catch panics
across a crossing boundary.

## Deploy (Onyx testnet)

```sh
# 1. Fund the keeper address from https://faucet.gno.land (network: Onyx)
#    Keeper: g1mqx99g0wjfzmjgpespuhq26kd59f3ek5cmkntv

# 2. Publish the realm
gnokey maketx addpkg \
  -pkgpath gno.land/r/g1mqx99g0wjfzmjgpespuhq26kd59f3ek5cmkntv/gnometavern \
  -pkgdir src/gnometavern \
  -gas-fee 1000000ugnot -gas-wanted 10000000 \
  -chainid onyx-1 \
  -remote https://rpc.onyx.testnets.gno.land:443 \
  tavernkeeper

# 3. Visit it
#    https://onyx.testnets.gno.land/r/g1mqx99g0wjfzmjgpespuhq26kd59f3ek5cmkntv/gnometavern
```

## Visit the tavern (gnokey)

```sh
R=gno.land/r/g1mqx99g0wjfzmjgpespuhq26kd59f3ek5cmkntv/gnometavern
RPC=https://rpc.onyx.testnets.gno.land:443

gnokey maketx call -pkgpath $R -func CheckIn -args "GandalfTheThirsty" \
  -gas-fee 1000000ugnot -gas-wanted 2000000 -chainid onyx-1 -remote $RPC mykey

gnokey maketx call -pkgpath $R -func OrderDrink -args 0 -send 50000ugnot \
  -gas-fee 1000000ugnot -gas-wanted 2000000 -chainid onyx-1 -remote $RPC mykey

gnokey maketx call -pkgpath $R -func Tip -send 100000ugnot \
  -gas-fee 1000000ugnot -gas-wanted 2000000 -chainid onyx-1 -remote $RPC mykey

gnokey maketx call -pkgpath $R -func CompleteQuest -args 0 \
  -gas-fee 1000000ugnot -gas-wanted 2000000 -chainid onyx-1 -remote $RPC mykey
```

## Sandbox note

This project was built inside a sandboxed VM whose egress goes through
an HTTP proxy that `gnokey`'s RPC client cannot use directly (it builds
its own transport and ignores proxy env vars, so direct connections
fail TLS interception). `tools/rpc_relay.py` is a tiny local relay that
forwards JSON-RPC traffic through the sanctioned proxy; point
`-remote http://127.0.0.1:26659` at it when working from that sandbox.
On a normal network, use the public RPC URL directly and ignore the
relay.

## Status

Code complete, lint clean, tests green. Deployment is one faucet drip
away (the Onyx faucet requires a human reCAPTCHA — see BUILD-LOG.md).
