# Venus - Smart Trading Bot

A bot written in Python that improves trading by executing **predefined
strategies** instead of leaving decisions to the moment. A user picks a
strategy and hands over a position; the bot manages it from there.

> Public write-up. The implementation is private.

## The concept

Manual trading fails in predictable ways: people take profits too early, hold
losses too long, and change their mind halfway through. A strategy fixes those
decisions in advance - where to take profit, in what proportions, and when to
cut a loss - and the bot's only job is to follow it without improvising.

Three presets, plus a fully custom option defined by five numbers:

| Strategy | Cut loss at | Target 1 | Target 2 | Target 3 |
|---|---|---|---|---|
| Conservative | -8%, trailing 10% below peak | +12%, close 40% | +25%, close 30% | +40%, close 20% |
| Standard | -12%, trailing 18% | +20%, close 25% | +45%, close 25% | +80%, close 20% |
| Aggressive | -25%, trailing 30% | +50%, close 20% | +120%, close 20% | +250%, close 20% |

**Protection only ever tightens.** The stop starts below the entry price,
follows the highest price seen at the strategy's trailing distance, and
ratchets upward each time a target is hit - after the first one it sits above
entry, so the position can no longer end at a loss. It never moves back down.
Once the last target is reached, the remainder rides the trend under the same
trailing stop.

Every action produces a permanent record: what closed, at what level against
the target, where protection now stands, and what happens next.

## Engineering notes

- **State is written before anything is reported back**, so an interruption
  mid-action can never repeat a close or weaken a stop.
- **The price feed is treated as untrusted.** An implausible quote is held
  until a second reading confirms it, after one corrupted value triggered a
  target at an absurd level.
- **16 test suites**, including a strict-compile gate and a test that diffs
  the two position-monitoring loops against each other so they cannot drift
  apart.

## Status

Not currently running. Source kept private.
