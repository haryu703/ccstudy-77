---
marp: true
paginate: true
---

# Bitcoin Cash Upgrade 2023-05-15

---

## 自己紹介

- [haryu703](https://twitter.com/haryu703)
- コインチェック株式会社 (2020~)

---

## 今回のアップグレードの概要

下記の４つの更新がある。

- CHIP-2021-01 Restrict Transaction Version
- CHIP-2021-01 Minimum Transaction Size
- CHIP-2022-02 CashTokens
- CHIP-2022-05 P2SH32

---

## [CHIP-2021-01 Restrict Transaction Version](https://gitlab.com/bitcoin.cash/chips/-/blob/3b0e5d55e1e139046794e850287b7acb795f4e66/CHIP-2021-01-Restrict%20Transaction%20Versions.md)

トランザクションのバージョンを1か2のみに制限する。

- 将来のアップグレードのためにルールを厳密にする
- 今までも1か2以外のバージョンはブロードキャストできなかった

---

## [CHIP-2021-01 Minimum Transaction Size](https://gitlab.com/bitcoin.cash/chips/-/blob/00e55fbfdaacf1436e455289086d9b4c6b3e7306/CHIP-2021-01-Allow%20Smaller%20Transactions.md)

トランザクションサイズの下限を100バイトから65バイトに緩和する。

- 64バイトのトランザクションはBitcoinのMerkle-treeにある脆弱性の影響を受ける
- 今までは100バイトが下限だったがcoinbaseトランザクションなど100バイト未満のトランザクションには需要があった

---

## [CHIP-2022-05 P2SH32](https://gitlab.com/0353F40E/p2sh32/-/blob/f58ecf835f58555c9087c53af25da92a0e74534c/CHIP-2022-05_Pay-to-Script-Hash-32_%28P2SH32%29_for_Bitcoin_Cash.md)

P2SHの新しい形式を追加する。

- 既存のP2SHは下記のようにHASH160が使われている
    `OP_HASH160 OP_DATA_20 hash160(redeem_script) OP_EQUAL`
- 今後色々なコントラクトを実装する際にHASH160だと衝突のおそれがある
- 下記のようにHASH256を代わりに使うフォーマットを追加する
    `OP_HASH256 OP_DATA_32 hash256(redeem_script) OP_EQUAL`

---

## [CHIP-2022-02 CashTokens](https://github.com/bitjson/cashtokens)

Bitcoin Cash上で任意のfungible tokenとnon-fungible tokenを扱えるようにする。

- 今までもOP_RETURNを利用する方法でのトークン表現はあったが、CashTokensはコンセンサスで検証されるBCH初めてのトークン

---

### CashTokensの応用

コントラクトの状態をNFTで表現することで、UTXOモデルでありながら高度なコントラクトを記述できるようになる。

- コントラクト間での状態の読み取り
- コントラクトの並列実行
  - アカウントモデルと違いトランザクションの順序はマイナーではなくユーザーが決める
  - コントラクトの実行時に消費するUTXOがユーザー間で競合することになる
  - 1つのコントラクトを複数のサブコントラクトに分割することで並列実行を実現する
- <https://github.com/bitjson/jedex>

---

### 概要

- Transaction outputに`token`フィールドを追加する。すべてのoutputは1つのNFTと１種類のFTを含めることができる
- Token inspection opcodesを追加する
- 署名時のシリアライズ方法の拡張
- `SIGHASH_UTXOS`を追加する
- CashAddrをトークンに対応させる
- BIP69をトークンに対応させる

---

### Token Category

- トークンは32バイトのToken Category IDにより識別される
- トークンのgenesis transactionのinputのうちvinが0のものを使うことができる

---

### Transactoin outputの拡張

今までの`satoshi_value`と`locking_bytecode`に加え下記の要素が加わる。

- Category ID (FTとNFT共通)
  - 32バイトのトークンの識別子
- Capability (NFTのみ)
  - NFTの種別。`none`、`mutable`、`minting`の３種類
  - `minting`は同じCategoryのNFTを複数発行できる
  - `mutable`は`commitment`を変更できる
- Commitment (NFTのみ)
  - 0-40バイトのNFTが持ってるCommitment
  - サイズは将来拡張される可能性がある
- Amount (FTのみ)
  - outputが持っているFTの数量
  - 最大値は9223372036854775807 (0xffff_ffff_ffff_ff7f)

---

#### Outputのフォーマット

Outputのフォーマットは下記のように拡張される。

```text
<satoshi_value> <token_prefix_and_locking_bytecode_length> [PREFIX_TOKEN <token_data>] <locking_bytecode>
```

- `PREFIX_TOKEN`は`0xef`で表現される

---

#### Token Data

`<token_data>`は下記のフォーマットに従う。

```text
<category_id> <token_bitfield> [nft_commitment_length nft_commitment] [ft_amount]
```

- <token_bitfield>
  - 2つの4bitのbitflag
  - 前半4bitは`HAS_NFT`や`HAS_AMOUNT`などあとに続くデータの構造を示す
  - 後半4bitはNFTのCapabilityを示す

---

### Token inspection opcodes

トークンの情報を取得する6つのopcodeが追加される。

- OP_UTXOTOKENCATEGORY (0xce)
- OP_UTXOTOKENCOMMITMENT (0xcf)
- OP_UTXOTOKENAMOUNT (0xd0)
- OP_OUTPUTTOKENCATEGORY (0xd1)
- OP_OUTPUTTOKENCOMMITMENT (0xd2)
- OP_OUTPUTTOKENAMOUNT (0xd3)

---

### 署名時のシリアライズ方法の拡張

- トランザクションのUTXOにトークンがある場合、署名対象にトークンの情報を含めなければならない
- 意図せずトークンの入ったUTXOを消費することが防がれる

---

### SIGHASH_UTXOS

署名対象に消費するUTXOを含めるための`SIGHASH_UTXOS` (`0x20`)が追加される。

- `hashUtxos`が`hashPrevouts`のあとに追加される
- `hashUtxos`は消費するすべてのUTXOのinputを結合したものに2回SHA256をかけたもの
- 複数の署名者が必要なトランザクションでは有効化することを推奨している
  - いわゆるSegwit Bugの対策になる

---

### CashAddrの拡張

トークンを受け取れることを示すためのアドレスタイプが追加される。

- P2PKHとP2SHに加えてToken-Aware P2PKHとToken-Aware P2SHが追加される

### BIP-0069の拡張

BIP-0069のトランザクションのoutputをソートするアルゴリズムをトークンに対応させる。

---

### 実際のトランザクション

- FT発行: <https://3xpl.com/bitcoin-cash/transaction/490db7049a0eb11594fa956db365408947cf5c054b60484743086f4577dc40e2>
- FT送信: <https://3xpl.com/bitcoin-cash/transaction/a59f373a0fd0cc4717e4d39d81230f8e4c0f4b922ae1b562c5d066900a615941>
- NFT発行: <https://3xpl.com/bitcoin-cash/transaction/78758fbd104c893e3b1a67a2e80694e6191d2bd09c421a7a5a825e1915e89e8f>
- NFT Mint: <https://3xpl.com/bitcoin-cash/transaction/6cc36e975c1730c1571ae248732ee6dca2aa864d61437e9697d38c1394e5f15c>

---

## 参考

- <https://upgradespecs.bitcoincashnode.org/2023-05-15-upgrade/>
