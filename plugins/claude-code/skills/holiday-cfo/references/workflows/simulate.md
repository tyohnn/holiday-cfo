# Simulate — what if

For "이 대출 받으면?", "집 사면?", "이 카드 다음 달에 다 갚으면?" — anything that is
not in the ledger yet.

Do NOT write speculative transactions and delete them. `holiday cashflow` takes the
hypotheticals directly and folds them into the runway, touching nothing:

```bash
holiday cashflow --until 2027-06-30 \
  --spend "2026-09-01 5000000 새 노트북" \
  --receive "2026-12-25 3000000 보너스" \
  --spend "2027-03-01 30000000 전세보증금"
```

`--spend` is money leaving, `--receive` is money arriving — no sign to guess. Both
repeat, so stack several and watch them interact. Each shows up as `가정: <label>`,
and the base ledger is untouched: re-run plain `holiday cashflow` to confirm.

For a recurring commitment (a new loan, a new 정기지출 or 정기수입), model the first few months
with several `--spend` lines rather than one, so the user sees the monthly bite.

## Taking something OUT of the projection — 상쇄 가정

Half of real what-ifs are subtractions: 구독을 해지하면, 이 일이 9월에 끝나면, 카드
생활비를 30만으로 묶으면. **Do not deregister the 정기지출 or edit the card to model
that.** Those are facts about today; a scenario that rewrites them corrupts the
ledger to answer a question.

Cancel it inside the projection instead — add the opposite assumption on the same
day, and say in the label that it is an offset:

```bash
holiday cashflow --until 2029-12-31 \
  --receive "2026-09-01 90000 가정:Cursor해지" \
  --receive "2026-10-01 36100 가정:KB변동30만캡"
```

Two consequences to hold:

- **Only the balance line is trustworthy afterwards.** An offset is an
  `assumption` item like any other, so 유입/유출 totals now double-count: the
  original outflow is still there and your inflow sits beside it. If you report
  monthly in/out (not just the runway), filter the offset pairs back out —
  which is why the label needs a marker you can grep (`가정:…해지`, `…제외`,
  `…캡`), not prose.
- **Offsets are assumptions, not corrections.** When the subscription is actually
  cancelled, close the 정기지출 with `--to` (see the ledger `AGENTS.md` §스케줄)
  and drop the offset. An offset that outlives the decision is a lie that
  compounds every month.

## Long horizons — script it

Past a few assumptions the command line stops being honest: nobody can re-derive
which of 40 `--spend` flags produced 2029-12. Once a scenario is worth deciding
on, write a script that builds the flags and shells the CLI:

```python
cmd = ["npx", "-y", "@holiday-cfo/cli@latest", "cashflow", "--until", UNTIL, "--json", *flags]
```

Keep three artifacts next to each other, and treat the first as the real one:

| | |
|---|---|
| `run_<scenario>.py` | the assumptions, as code — salary, 요율, payoff dates, offsets |
| `<scenario>-raw.json` | the CLI's output, unedited |
| `<scenario>.md` | what you tell the user |

The script is the point: a scenario nobody can re-run is an opinion, and next
month the user will ask "이거 그때 그 가정 그대로야?" Variants (with/without a
raise, ICL monthly vs lump) belong in the **same** script behind a flag, so two
numbers differ by exactly one assumption.

Constants that are 관측값 — 간이세액표 rows, 보험요율 caps, a lender's quoted
payment — get a comment naming the source and date, same rule as the ledger:
never a computed rate, never a remembered one.

## The answer is three numbers

Whatever the horizon, report these and stop:

1. **첫 부족** — date and amount of the first ⚠, or "부족 없음".
2. **월중최저** — the lowest the balance gets and when. A month can end fine and
   still fail on the 22nd.
3. **기말잔액** at the horizon.

Then one line on what moves the first number. That is the decision; the table is
supporting material.

## Two edges of the projection

- **주말·공휴일 롤이 없다.** A `--payment-day 1` landing on Saturday is projected
  on the 1st; the bank moves it. Model the slip with `--spend` on the real date —
  do not change the card's contract day to make the projection prettier.
- **전망은 오늘부터다.** `cashflow` has no `--as-of`, and every scheduled row —
  card bill, 할부, 정기지출, loan — whose payment date is today or earlier is
  treated as settled and drops out. Assumptions are the exception: `--spend` /
  `--receive` count from today inclusive. That asymmetry is what lets you model a
  slipped payment — put today's dropped bill back with `--spend` on the date the
  bank will really take it. Once it actually moves, record it (`txn add` /
  `loan pay`) and remove the assumption.
