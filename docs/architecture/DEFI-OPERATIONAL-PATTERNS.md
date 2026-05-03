# PRMX DeFi Operational Patterns — Bittensor/TaoFi 実装から学ぶ運用指針

> **Source**: Bittensor (Subtensor + Frontier EVM + Hyperlane) および TaoFi の公開実装調査
> **Scope**: PRMX の Hyperlane Warp Route / ICA 運用と、将来拡張時に維持すべき DeFi 運用パターン集
> **PRMX baseline**: `pallet-assets(1)` is the settlement-balance source of truth; `VAULT_REPORTER_TRANSPORT=hyperlane` is the active yield-report transport; the Morpho Blue testnet strategy is deployed. Borrower automation is testnet-only and off-book — not a settlement ledger and not a canonical capital path.

本ドキュメントは「設計決定書」ではなく「推奨運用指針」です。各項目に **採用レベル**(絶対採用 / 強く推奨 / 段階採用 / 参考 / 不採用)を付してあります。

---

## 採用優先度サマリー

| # | パターン | PRMX への適用先 | 採用レベル |
|---|----------|----------------|-----------|
| 1 | Rebalancer executor 分離(owner ≠ executor) | Hyperlane ICA / `PolicyVaultManager` | **絶対採用** |
| 2 | Per-bridge / per-target whitelist | Hyperlane ICA command targets | **絶対採用** |
| 3 | HSM / MPC 鍵管理 | Executor EOA, Council Safe | **絶対採用** |
| 4 | `rebalance(amounts[])` バッチパターン | PolicyVaultManager の再配分処理 | **絶対採用** |
| 5 | Owner / Rebalancer 権限分割 | Council Safe vs Operator | **強く推奨** |
| 6 | XERC20 mint rate limits | 将来の native PRMX 拡張時 | **段階採用** |
| 7 | TVL cap at launch | Reserve Layer の初期上限 | **段階採用** |
| 8 | Yield strategy externalization | Morpho Blue testnet strategy + future layers | **参考** |
| 9 | ICA single-call executor discipline | Rebalance / reserve / strategy operations | **絶対採用(single-call)** |
| 10 | Bittensor 固有プリコンパイル活用 | (該当なし) | **不採用** |

---

## 1. Rebalancer executor 分離パターン【絶対採用】

### Bittensor 実装

Abacus Works(Hyperlane 運営元)が運用する USDC Warp Route では、全チェーン共通で **単一の EOA アドレス**(`0xa394...8FE3` 等の運用用ホットウォレット)が `allowedRebalancers` に登録されている。この EOA は:

- **owner ではない**(owner は Safe マルチシグ)
- **rebalance 以外の権限を持たない**(admin 操作不可)
- 毎日複数回の自動再配分を実行

### PRMX への適用

```
Council Safe (3-of-5 multisig)       ← Owner(admin 操作: addRebalancer, addBridge)
    │
    └─ IcaCommandRouter               ← Base 側 allowlisted executor
         │
         └─ PolicyVaultManager        ← Rebalance / reserve operation target

PRMX rebalancer executor              ← PRMX 側から Hyperlane ICA を発火
    └─ InterchainAccountRouter → Base ICA proxy → IcaCommandRouter
```

**重要な分離原則**:
- **Owner 鍵**: Council Safe。addRebalancer / addBridge / pauseRouter 等の admin 操作専用。通常運用では使わない(数ヶ月単位で動かさない)
- **Executor 鍵**: Operator EOA または KMS/HSM。PRMX 側の ICA submission 専用。Base の admin 権限なし
- **緊急時**: Operator 鍵が漏洩しても Council Safe から `removeRebalancer` で即座に隔離可能

### なぜ PRMX に効くのか

PRMX は毎日複数回(想定: 1日3-10回)の再配分が必要になる。運用頻度が高いほど鍵露出リスクも高まるため、**最悪ケースの被害を ICA 経由の allowlisted operation に限定する**のが生存戦略として必須。Bittensor の実運用実績が、この分離で運用コスト・セキュリティ両立が可能であることを示している。

---

## 2. Per-bridge / per-target Whitelist パターン【絶対採用】

### Bittensor 実装

`MovableCollateralRouter.allowedRebalancingBridges[domain]` で、**ドメイン(チェーン)ごとに使用可能な `ITokenBridge` 実装を個別登録**する方式。ランダムな bridge コントラクトを rebalancer が指定することはできない。

### PRMX への適用

- **Base(domain=84532)** 向け operation: `IcaCommandRouter` と `PolicyVaultManager` / `PolicyVault` targets を明示登録
- **同一チェーン再配分**: `domain = baseDomain` を指定し、登録済み target 経由でのみ実行
- **Council Safe による登録**: 新規 PolicyVault デプロイのたびに allowlist 追加を 3-of-5 で承認

### なぜ PRMX に効くのか

Rebalancer EOA が侵害されても、攻撃者は「事前に Council が承認した bridge アドレス」にしか資金を動かせない。**承認リストが明示的 allow-list であるため、任意の attacker-controlled コントラクトへの流出が構造的に不可能**。これは Bittensor が USDC TVL を守るために選んだ最低防衛線であり、PRMX にとっても同等の価値を持つ。

---

## 3. HSM / MPC 鍵管理【絶対採用】

### Bittensor 実装(公開情報から推定)

Abacus Works の Rebalancer EOA は、プライベートキーを平文で運用することはなく、クラウド HSM(AWS KMS / GCP Cloud KMS 等)または MPC(Fireblocks / Copper 等)で署名を分離している。

### PRMX への適用

**フェーズ別採用**:

1. **テストネット初期**: 運用簡便性優先で `ETH_PRIVATE_KEY` 環境変数(`~/.prmx/local-secrets.env`)+ DO サーバー経由で許容
2. **テストネット後期 / メインネット前**: Operator EOA を GCP Cloud KMS 管理に移行
3. **メインネット(将来)**: Council Safe は Gnosis Safe + Ledger ハードウェア署名、Operator EOA は HSM または MPC 必須

### なぜ PRMX に効くのか

Rebalancer EOA は毎日稼働するため、サーバー侵害時の鍵漏洩リスクが構造的に存在する。HSM/MPC はコード実行環境から秘密鍵を隔離するため、サーバーが完全に乗っ取られても**鍵そのものは攻撃者の手に渡らない**(署名要求は呼べるが、使途は per-bridge whitelist で制限される)。

---

## 4. `rebalance(amounts[])` バッチパターン【絶対採用】

### Bittensor 実装(TaoFi / TAOStaker)

TAOStaker は複数の staking validator への委任を `rebalance(uint256[] amounts)` という単一トランザクションで一括調整する。各 validator ごとに金額を指定し、不足分は追加委任、過剰分は回収。

### PRMX への適用

実行パスは `oracle-service` rebalancer executor -> PRMX `InterchainAccountRouter`
-> Base ICA proxy -> `IcaCommandRouter` -> `PolicyVaultManager`。直接 Base write は運用違反として扱う。

`PolicyVaultManager.rebalance(policyIds[], amounts[])`:

```solidity
function rebalance(
    bytes32[] calldata policyIds,
    uint256[] calldata amounts  // Reserve 層から各 PolicyVault へ送る増減額
) external onlyOperator {
    // 1. 各 policy について amounts[i] - currentDeployed[i] を計算
    // 2. 正: HypERC20Collateral.rebalance(baseDomain, diff, PolicyVaultBridge) で追加デプロイ
    // 3. 負: PolicyVault.withdrawToReserve(|diff|) で Reserve に引き戻し
    // 4. 最後に BackingOracle に集約した deployedTotal を更新
}
```

### なぜ PRMX に効くのか

- **Gas 効率**: N 個の policy を 1 tx で処理 → Base 上のガスコスト大幅削減
- **原子性**: 中途半端な再配分状態(一部だけ動いた)を避ける。すべて成功かすべて失敗
- **監査性**: 1 tx = 1 リバランスサイクル。イベントログから「その時点の意図」が明確
- **Invariant 検証**: バッチ完了後に Reserve + Σ(deployed) = Σ(policy capital) を 1 回チェックするだけで済む
- **Yield 分離**: rebalance credit/debit と funding は yield ではなく capital movement として扱う

---

## 5. Owner / Rebalancer 権限分割【強く推奨】

### Bittensor 実装

- **Owner**: Multi-sig Safe(Abacus Works チーム 3-of-5)
  - addRebalancer, removeRebalancer, addBridge, pauseRouter, upgradeImplementation
- **Rebalancer**: ホット EOA(単独署名)
  - rebalance(domain, amount, bridge) のみ

### PRMX への適用

PRMX 既存の Council Governance(2/3 投票: Athena / Apollo / Hermes)を **Owner 役**として再利用し、DAO Operator(Alice)または新規 Operator EOA を **Rebalancer 役**として割り当てる。

```
Council Majority Origin (2/3)
  ├─ addRebalancer(operator)
  ├─ addBridge(domain, policyVault)
  └─ removeRebalancer(operator)   ← 緊急時 kill-switch

Operator Origin (Alice または HSM/KMS EOA)
  └─ PRMX ICA submission -> Base IcaCommandRouter -> PolicyVaultManager.rebalance(policyIds, amounts)
```

### なぜ PRMX に効くのか

PRMX はすでに Council / operator / reporter 系の権限分離パターンを採用済み。Hyperlane 側にも同じ 2 層ガバナンスを拡張することで、**「オンチェーン統治の一貫性」**が保たれる。Bittensor が別途 Safe を立てているのと対照的に、PRMX は Council Safe を再利用できるので運用コストが低い。

---

## 6. XERC20 Mint Rate Limits【段階採用】

### Bittensor 実装(TaoFi taoUSD)

TaoFi の `taoUSD` は **XERC20 + Lockbox パターン**を採用し、各 minter ごとに `maxLimit` と `rate per second` を個別設定。1 日あたりの最大発行量を構造的に制限している。

```solidity
setLimits(minter, mintLimit, burnLimit)
// 時間経過で limit が回復する leaky bucket 方式
```

### PRMX への適用タイミング

**現時点では不要**(HypERC20Collateral は担保ロック方式で、native mint ではない)。ただし、将来以下のシナリオで採用検討:

- **Plan A 移行時**: PRMX が native トークン(mUSDC の再定義 / 独自 stable 発行)を導入した場合
- **マルチチェーン拡張時**: Base 以外のチェーン(Arbitrum / OP mainnet 等)に Warp Route を拡張し、各 bridge を独立 minter として扱う場合

### なぜ PRMX に将来効くのか

Rebalancer 鍵 + bridge 経路の両方が侵害された最悪ケースでも、**1 日あたりの流出額が rate limit で上限される**。これは「多層防御の最後の層」として機能する。Bittensor が TaoFi で採用している事実は、実運用で価値が証明されたパターンであることを示す。

---

## 7. TVL Cap at Launch【段階採用】

### Bittensor 実装

TaoFi は USDC Warp Route の**ローンチ時に 10M USDC の TVL キャップ**を設定。徐々に緩和していく段階的リリース戦略を採用している。

### PRMX への適用

```solidity
// PrmxReserveCap.sol(新規、オプション)
uint256 public maxReserveTVL = 1_000_000e6;  // 初期 1M USDC

function _preDeposit(uint256 amount) internal view {
    require(
        HypERC20Collateral.wrappedToken.balanceOf(address(this)) + amount
        <= maxReserveTVL,
        "Reserve cap exceeded"
    );
    // Council Majority で maxReserveTVL を段階的に緩和
}
```

### なぜ PRMX に将来効くのか

- **ブロー半径の限定**: 初期バグ / 設計ミスが露見した際の損失上限を明示
- **段階的信頼構築**: 1M → 10M → 100M と TVL を緩めることで、各段階での監視・監査機会を確保
- **規制対応**: 将来の規制当局との対話で「リスク限定設計」の根拠として提示可能

ただし PRMX の policy-backed 設計では **Reserve は policy 総資本の従属変数**であり、キャップを policy 側に設けた方が自然な場合もある。採用するかどうかは Reserve 設計確定後に判断。

---

## 8. Yield Strategy Externalization【参考】

### Bittensor 実装

TaoFi は yield 源を:

- **Sturdy Finance** の isolated lending pair(taoUSD / USDC)→ 低リスク基礎利回り
- **Subnet 10 AI strategies** → 高リスク高利回り(AI 運用型)

の 2 系統に切り出し、ユーザーが選択できる設計。TAOStaker はさらに複数 validator への委任を抽象化。

### PRMX への適用

現時点の testnet では、MockLendingVault は loss-mode harness として残しつつ、Morpho Blue testnet strategy が deployed/verified 済み。`VAULT_REPORTER_TRANSPORT=hyperlane` 経由で PRMX 側の vault snapshot へ届ける運用対象。

将来的には:

- **Layer 1** = Aave V3 aUSDC(低リスク)
- **Layer 2** = Compound V3 cUSDC(中リスク)
- **Layer 3** = Morpho Blue 独自市場(高リスク、testnet verified)

という多層戦略に拡張した際、Bittensor の「ユーザー選択式 yield」ではなく **PRMX 側が単一ポリシーで最適配分する**方向が自然(parametric 保険の性質上、LP はリスク調整後利回りを一元化したい)。

Borrower automation は testnet-only/off-book。Morpho utilization と strategy P/L
を作るための検証補助であり、ユーザー向け貸付台帳、settlement-balance SSoT、
または canonical capital path ではない。

### なぜ PRMX には限定適用なのか

PRMX と TaoFi では **yield 設計思想が根本的に違う**:
- TaoFi: ユーザーが AI サブネットを選んで利回りを追う(アクティブ運用)
- PRMX: LP は parametric 保険料収入が主、yield は sub-layer(パッシブ運用)

したがって「yield 戦略を外部化する」という思想は参考に留め、PRMX 独自の階層設計を優先する。特に initial funding / rebalance / over-backed surplus は yield ではなく capital movement として quarantine し、zero-baseline report は matching ack が出るまで yield credit しない。`record_vault_funded` が baseline calibration の起点であり、rebalance-in は `uncalibrated_or_unknown` / `already_funded` / `real_drawdown_candidate` に分類して、実 drawdown だけを自動 top-up 対象にする。Triggered payout 時の negative strategy P/L は DAO backstop で補填し、`pallet-assets(1)` の未担保 mint で隠さない。

---

## 9. ICA single-call executor discipline【絶対採用(single-call)】

### Bittensor 実装(該当なし)

Bittensor 自体は ICA ベースのクロスチェーン governance 多用パターンは確認できず。Subnet を跨ぐ処理は主にオフチェーン。

### PRMX への適用

Hyperlane ICA は Base operation path として採用済み。ただし、**横断的「アトミック多段呼び出し」は PRMX では採用しない**:

- Rebalancer executor、reserve return、strategy operation は PRMX 側から ICA single-call として発火する
- Base `PolicyVaultManager` / `VaultFactory` への直接 write は禁止
- `executeSingleCall` 中心で十分。`executeMultiCall` を使うケースが現状ない

### なぜ multi-call は不採用なのか

複数 tx のアトミック性が必要な場面が PRMX の運用モデルに存在しない(各 rebalance / invariant 更新は独立)。複雑度を増やすだけなので導入しない。

---

## 10. Bittensor 固有プリコンパイル活用【不採用】

### Bittensor 実装

Bittensor は `pallet_subtensor` とのブリッジのため、独自プリコンパイル(Substrate pallet → EVM 呼び出し)を多用。TaoFi は `0x0000...0802` 等の Subtensor precompile を介して staking 情報にアクセス。

### PRMX への適用

**Bittensor 固有パターンとしては該当なし**。PRMX は PRMX EVM の限定 precompile
(`0x0800` / `0x0801` / `0x0802` / `0x0807` / `0x0808`)を canonical bridge / exit / yield report / 読み取り / ユーザーアクションに使うが、
Base から Substrate 側を直接呼ぶ運用は採らない(Hyperlane メッセージング経由で十分)。

### なぜ不採用なのか

- PRMX EVM precompile は狭い settlement surface に限定する
- Bittensor の Subtensor precompile 活用を Base 側 operation に拡張する必要はない

---

## 情報ギャップ(公開情報から得られなかった事項)

**honest disclosure**: 以下は Bittensor / TaoFi の公開 GitHub / ブログから**確認できなかった**ため、PRMX 採用時は独自設計または追加調査が必要:

1. **Rebalancer 発動閾値**: 「どのくらいの乖離で rebalance を発動するか」の具体数値(例: Reserve が targetReserveRatio の ±5% を超えたら発動、等)
2. **Rebalance 発動頻度**: 1 日 3 回 / 毎時 / イベント駆動、等の具体ポリシー
3. **Operator インフラ**: クラウドプロバイダ、冗長化構成、監視スタック
4. **インシデント履歴**: 過去の rebalancer 停止 / 鍵ローテーション事例
5. **HSM / MPC の具体ベンダー**: Fireblocks / Copper / AWS KMS / 自前実装のどれか

PRMX 実装時の推奨アプローチ:
- **閾値・頻度**: 自前の Reserve 設計(Y3 Split Collateral)から独自に導出
- **インフラ**: DigitalOcean + GCP KMS の既存パターン流用
- **HSM**: メインネット直前に Fireblocks または GCP Cloud KMS のいずれかを採用

---

## 採用ロードマップ

| フェーズ | 採用項目 | タイミング |
|---------|---------|-----------|
| 現行 testnet | #1, #2, #4, #5, #8, #9(single-call) | Hyperlane-only route / Morpho testnet verified |
| Pre-mainnet hardening | #3(初期は env 鍵、後に KMS)、#7(Reserve cap) | TVL 拡大前 |
| 将来拡張 | #6(XERC20 rate limits) | Plan A(native token)検討時 |

---

## 参照

- [m72 — pallet-assets canonical path decision](/docs/hyperlane-migration/m72-pallet-assets-hyperlane-canonical-path-decision) — `pallet-assets(1)` を SSoT とする canonical path 設計
- [m75 — ICA Yield Command Bus](/docs/hyperlane-migration/m75-ica-yield-command-bus) — ICA + yield コマンドバス設計
- [hyperlane-monorepo/MovableCollateralRouter.sol](https://github.com/hyperlane-xyz/hyperlane-monorepo) — 標準実装(unmodified)
- [TaoFi taoUSD contracts](https://github.com/taofi) — XERC20 + Lockbox 参考実装
- [Abacus Works USDC Warp Route config](https://github.com/hyperlane-xyz/hyperlane-registry) — Rebalancer EOA 運用の実例
