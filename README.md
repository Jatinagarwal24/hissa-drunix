# HISSA

**Tokenised co-lending settlement utility on NPCI DRUNIX.**

Citi x NPCI DRUNIX Hackathon 2026 (Innovation Carnival, challenge CHL-7007). Track: Financial Inclusion.

> Status: proposal stage. Idea submitted 24 Oct 2026. Build runs 10 to 22 Nov 2026. Everything under "Architecture" below is planned unless marked otherwise.

## Team

**Prime Minds**, Indian Institute of Technology Gandhinagar

| Member | Role |
| --- | --- |
| Yug Desai | Team lead |
| Anirudh Mittal | Member |
| Jatin Agarwal | Member |
| Harshil Shah | Member |

Pitch deck: [docs/HISSA_Pitch_Deck.pdf](docs/HISSA_Pitch_Deck.pdf)

## What it is

In co-lending a bank and an NBFC fund each loan together, typically 80:20. Every EMI has to be split, routed through escrow, and both lenders must carry the same SMA/NPA classification. Today that runs on batch files and spreadsheets, and the RBI Co-Lending Directions 2025 (effective 1 Jan 2026) turn it into a compliance deadline.

HISSA ("hissa" means share) puts each co-lent loan on a DRUNIX network operated by NPCI. The loan is cut into participation units held by the two lenders in their agreed ratio. Collections, splits, classification and transfers settle as shared state that neither lender owns alone and the regulator can read. Think of it as NACH for co-lending.

## Lifecycle on-chain

1. **CLA registry.** The master agreement (ratio, blended rate, fee split, DLG cap, escrow reference) is registered once. Terms live in a private data collection shared by the two lenders and the escrow bank.
2. **Loan tokenisation.** On disbursal, chaincode mints units (8,000 bank / 2,000 NBFC per Rs 10,000). The 10% retention floor is enforced in code. Borrower PII never touches the shared channel.
3. **Collections and settlement.** UPI AutoPay / NACH events credit a tokenised escrow ledger. The settlement contract applies the waterfall and credits each lender at T+0, netting to one RTGS instruction per bank per day.
4. **Classification.** DPD/SMA/NPA is one shared state with a worst-of rule. An update reaches both books in minutes.
5. **Transfer of exposure.** Selling a share to another bank is a unit transfer with consent (TLE). Later extends to direct assignment and PTC securitisation.
6. **Supervision.** RBI's read-only node runs SQL (YugabyteDB) over ledger state.

## Architecture (planned)

| Layer | Choice |
| --- | --- |
| Ledger | DRUNIX (NPCI's Hyperledger Fabric fork), 5 orgs: originating RE, partner RE, escrow bank, NPCI operator/orderer, RBI observer |
| Chaincode | Go, fabric-contract-api-go: `cla`, `participation-token`, `escrow`, `settlement-waterfall`, `classification`, `tle`, `dlg` |
| State DB | YugabyteDB for regulator and escrow SQL reporting |
| Privacy | Per-pair private data collections via the KeyDB transient store |
| Scaling | Lite Peers scaled for month-start EMI peaks; stateless Validation Service scaled for NACH presentation batches |
| Gateway | Node service on the Fabric Gateway SDK: REST APIs, collection webhooks, chaincode event listeners |
| Deployment | Falcon Helm charts on Kubernetes |

## Why DRUNIX

- **Lite / Committing Peer separation.** Endorsement scales independently for the 1st to 10th EMI peak without replicating ledger storage.
- **Stateless Validation Service.** Absorbs NACH presentation bursts in parallel.
- **YugabyteDB.** Supervisory queries straight from ledger state. LevelDB cannot query, CouchDB does not aggregate at this scale.
- **KeyDB transient store.** Every loan write carries private data between two lenders; the shared transient store cuts latency on exactly that path.
- **Multi-org design.** Mirrors how NPCI runs NACH: neutral operator, members keep their own books.

## Build plan (10 to 22 Nov)

| Days | Work |
| --- | --- |
| 1 to 2 | Four-org DRUNIX test network |
| 3 to 6 | Core chaincode and unit tests |
| 7 to 9 | Gateway, simulated AutoPay/NACH webhook, four dashboards (NBFC, bank, escrow, RBI SQL) |
| 10 to 11 | Private data collections, load demo on 2 Lite Peers + 2 Validation Service instances, Falcon deploy |
| 12 | Demo video, docs, final code submission |

Demo flow: onboard a CLA, disburse a Rs 5 lakh loan 80:20, an EMI arrives and splits instantly, DPD slips and SMA-1 propagates to both books, 30% is sold to a second bank, RBI runs its query in SQL.

## Repository layout

```
chaincode/   Go contracts (from 10 Nov)
gateway/     Node gateway and dashboards (from 10 Nov)
network/     DRUNIX test-network config and deploy scripts (from 10 Nov)
docs/        Pitch deck and architecture notes
```

## References

- [npci/drunix](https://github.com/npci/drunix) and its [architecture notes](https://github.com/npci/drunix/blob/main/docs/drunix-arch.md)
- [npci/falcon](https://github.com/npci/falcon)
- [RBI Co-Lending Arrangements Directions, 2025](https://www.mondaq.com/india/fund-finance/1679822/rbis-2025-co-lending-regime-balancing-prudence-growth-and-protection)
- [Crisil: NBFC co-lending AUM nearing Rs 1 lakh crore](https://www.crisilratings.com/en/home/newsroom/press-releases/2024/04/nbfc-co-lending-aum-closing-in-on-rs-1-lakh-crore-mark.html)

## License

Apache-2.0
