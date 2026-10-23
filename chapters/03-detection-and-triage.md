# Detection and Triage

Most protocols learn about an exploit in one of three ways: their own monitoring fires, an external security firm or researcher posts an alert, or users complain. The first is the only one you control and usually the fastest. Once an alert arrives, triage must answer a handful of questions within minutes, with partial information and the attacker possibly still active. This chapter covers both.

## Detection sources

| Source | Strength | Weakness |
|---|---|---|
| Invariant monitors (balances, supply, collateral ratios) | Catch unknown bug classes by their effect | Fire after loss starts |
| Admin and role-change monitors | Catch key compromise and governance abuse early | Need allowlists of expected actions |
| Mempool monitoring for suspicious calls | Can fire before inclusion | Private order flow bypasses the public mempool |
| Attack-pattern detection services (e.g. Forta network bots, Hypernative, Hexagate, BlockSec Phalcon) | Specialised heuristics, sometimes pre-exploit detection of attacker contract deployment | Vendor-specific coverage; false positives |
| Simulation-based alerting (e.g. Tenderly alerts) | Rich context per transaction | Cost and setup |
| Community, researchers, other protocols | Broad coverage | Unstructured, noisy, slow |

A widely observed pattern is that attackers often deploy an exploit contract before using it. Detection systems that decompile and simulate newly deployed contracts against known protocols have, according to vendor reports, sometimes flagged attacks in this window. Treat such claims as vendor-reported and test them against your own historical incidents.

### What to monitor

At minimum: total value locked per pool, outflow per block and per hour, share price or exchange rate of vault tokens, oracle prices against references, role grants and ownership transfers, upgrades and implementation changes, pause state, and large approvals to unknown contracts from your treasury.

```yaml
# Illustrative invariant rule (pseudo-configuration)
- name: vault-share-price-drop
  every: block
  check: share_price(now) < 0.97 * share_price(now - 10 blocks)
  severity: critical
  route: pagerduty:onchain-oncall
```

## Triage questions

Within the first minutes, answer in order:

1. **Is it real?** Confirm with a second source: block explorer, your own node, the transaction trace.
2. **Is it ongoing?** Are more attack transactions arriving? Is the vulnerable path still open?
3. **What is at risk now?** Remaining funds in affected contracts, other deployments of the same code on other chains, integrations.
4. **What category?** Contract bug, key compromise, front-end, governance, oracle, third-party.
5. **Can we stop it?** Which pause or guardian action applies, and who must sign?
6. **Who needs to know immediately?** Signers, SEAL 911, integrators, stablecoin issuers if attacker holds their tokens.

Write answers in the war-room channel with timestamps. That log becomes the backbone of the post-mortem.

## Understanding the exploit transaction

Speed matters, but so does avoiding a pause or fix based on the wrong theory. Tools for quick analysis:

- **Trace viewers**: Phalcon Explorer, Tenderly, and Etherscan's trace views show call trees, balance changes and storage writes.
- **Local replay**: `cast run <txhash> --rpc-url <archive-rpc>` re-executes the transaction with a trace on your machine.
- **Fork reproduction**: Foundry tests with `vm.createSelectFork(rpc, block)` let you reproduce the exploit just before the attack block and test whether your pause or patch stops it.

```bash
# Re-execute an incident transaction with a full trace (archive RPC needed)
cast run 0x<txhash> --rpc-url $ARCHIVE_RPC --quick
# Decode an unknown selector seen in the trace
cast 4byte 0x<selector>
```

AI assistants can help summarise a long trace or propose a hypothesis about the root cause. Treat that output as a hypothesis only: confirm it by reproducing the exploit on a fork and showing that the proposed fix makes the reproduction fail. The fork test is the checker; the model is not.

## Severity and escalation

| Severity | Example | Response |
|---|---|---|
| Critical | Ongoing loss of user funds | Immediate pause, all hands, SEAL 911 |
| High | Vulnerability confirmed, not yet exploited | Private fix, consider pause, limited circle |
| Medium | Loss limited and stopped, or dependency issue | War room, scheduled fix |
| Low | Suspicious activity, unconfirmed | Investigate during working hours |

Confirmed-but-unexploited vulnerabilities need special care. Pausing or deploying a fix may itself reveal the bug to watchers; coordinate with responders on whether a quiet fix, a pause with a vague message, or a whitehat rescue is safest.

## Worked example

*Execution status: the commands, code and numbers in this example were not executed in this book's CI; they are illustrative. Verify locally on a devnet, testnet or fork before relying on them.*

Triage a historical exploit as if it were live, on a fork.

1. Pick a well-documented past exploit with a public post-mortem (for example, the Euler Finance exploit in March 2023).
2. Without reading the post-mortem, open the main exploit transaction in a trace viewer and write a one-paragraph hypothesis.
3. Create a Foundry test that forks mainnet at the block before the exploit and replays the attacker's calls from a test contract.
4. Answer the six triage questions using only on-chain data at that block: what else was at risk, which roles could pause, and which addresses were involved.
5. Compare your notes with the published post-mortem and record what you missed and how long each step took.

## Common pitfalls

- Pausing based on a single unverified social media alert, or not pausing while waiting for certainty.
- Only monitoring your main chain deployment.
- Alert routing to a channel nobody watches at night.
- Accepting an AI or tool explanation of the exploit without reproducing it.
- Losing the timeline because discussion happened in calls with no notes.

## Exercises

1. Write five invariant monitors for a protocol you know and specify thresholds and routes.
2. Reproduce one historical exploit in a Foundry fork test and add a test proving a pause at block N−1 would have stopped it.
3. Run a triage drill with a teammate who plays the attacker, injecting transactions on an Anvil fork while you respond.

## Further reading

- Foundry book, `cast run` and fork testing: <https://book.getfoundry.sh>
- Forta documentation: <https://docs.forta.network>
- BlockSec Phalcon explorer: <https://phalcon.blocksec.com>
- Euler Finance post-mortem (2023) and other team-published post-mortems
- Related books: **onchain-malware-analysis**, **defi-economic-attack-simulation**, **oracle-security-engineering**
