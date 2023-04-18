---
title: "Manifoldで作成したClaim PageをNode.js(TypeScript)で直接Mintする方法"
emoji: "🌱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Solidity", "Manifold", "Node.js", "TypeScript"]
published: true
published_at: 2023-04-18 23:25
---

## はじめに
この記事では **Manifold** の拡張機能である **Claim Page** から実行できる Mint のトランザクションを、Node.js(TypeScript) で実装したプログラムから直接実行する方法を紹介します。

※Manifold を使った具体的なNFTコントラクトの作成・メインネットワークへの発行方法は省略します。また、特定のウォレットアドレスのみ Mint を許可された Limited Edition (Allow List 形式)での検証はできていないため、本記事では取り上げません。

## 検証環境
- Node.js v16 系
- TypeScript v4.8
- ethers v5.7

### Manifold とは何か
Manifold はのNFT発行を独自コントラクトとしてWeb上で公開できるサービス(DApp)です。
ERC721, ERC1155 と一般的な規格に対応しており、その簡単さから海外で流行し、続々と日本の著名NFTクリエイター、ファウンダーが導入するようになりました。

https://manifold.xyz

Manifoldで作られたNFTコレクションは、**Creator Contract** と呼ばれる ERC721, ERC1155 を拡張したアップグレード可能なスマートコントラクトです。

Creator Contract のソースコードは以下のリポジトリに公開されています。
https://github.com/manifoldxyz/creator-core-solidity

- [ERC721 の Creator Contract のソースコード](https://github.com/manifoldxyz/creator-core-solidity/blob/main/contracts/core/ERC721CreatorCore.sol)
- [ERC1155 の Creator Contract のソースコード](https://github.com/manifoldxyz/creator-core-solidity/blob/main/contracts/core/ERC1155CreatorCore.sol)

ソースコードから気付くべきなのが、**Creator Contract 自身には第三者から直接NFTを発行(Mint)する機能が無い**という点です。Creator Contract は、クリエイター自身かクリエイターが直接指定したアドレスに向けて送付(Airdrop)することのみ可能です。

第三者からMintしてもらうようにするには、Creator Contract が搭載している拡張機能の登録を行う必要があります。
その拡張機能が **Claim Page** であり、拡張機能を Manifold内では **Apps** と呼んでいます。

### Claim Page とは何か
Claim Page はコレクター(第三者)にクリエイターが発行するトークンをMintできるようにしてくれる Manifold の拡張機能です。

※ 以下はClaim Page のサンプルです(メインネットのものなので普通にトランザクションを実行するとETHが消費されるのでご注意を)
https://app.manifold.xyz/c/bruceleeofficial

各拡張機能のソースコード以下のリポジトリに公開されています。
https://github.com/manifoldxyz/creator-core-extensions-solidity

Claim Page で実行するスマートコントラクトは、ERC721, ERC1155 ごとに別々で用意されており、それぞれ **`ERC721LazyPayableClaim`**, **`ERC1155LazyPayableClaim`** として定義されています。(以降まとめてLazyClaimと呼びます)
https://github.com/manifoldxyz/creator-core-extensions-solidity/tree/main/contracts/manifold/lazyclaim

Lazy Claim のコントラクトアドレスは[README.md](https://github.com/manifoldxyz/creator-core-extensions-solidity#manifold-lazy-claim)にそれぞれ記載されています。
> ERC721 Mainnet
>```
>0xa46f952645D4DeEc07A7Cd98D1Ec9EC888d4b61E
>```
> ERC1155 Mainnet
>```
>0x44e94034AFcE2Dd3CD5Eb62528f239686Fc8f162
>```

ここで注意するべき点なのは、**コレクター(第三者)が Mint トランザクションを行う送信先(to)は、Lazy Claim のスマートコントラクト宛になること**です。
ですので、本記事の表題の通り、プログラムから Mint を実行するには、**送信先を Creator Contract ではなく Lazy Claim 宛に送信する必要があります。**


Creator Contract をデプロイ後、Manifold Studio(クリエイター側の管理画面)内にある Apps ページから設定する導線があります。
クリエイターは第三者にMintする上で支払ってもらう金額(フリーミントにするなら0ETH)、発行枚数、slug などを入力し、自身の Creator Contract との紐付けを行うトランザクションを実行します。

このとき発生するトランザクションは順に2個存在します

1. [registerExtension](https://docs.manifold.xyz/v/manifold-for-developers/smart-contracts/manifold-creator/prior-versions/2.0.x/extensions/extensions-functions#registerextension)
   Creator Contract と Lazy Claim 間の紐付けを行います。このトランザクション関数は Creator Contract 側に定義されています。
2. [initializeClaim](https://github.com/manifoldxyz/creator-core-extensions-solidity/blob/main/contracts/manifold/lazyclaim/IERC1155LazyPayableClaim.sol#L48)
   Claim Page で Mint できるトークン情報を Lazy Claim 側に保存します。このトランザクション関数は Lazy Claim 側に定義されています。

## ethers.js を使った Claim App で公開されているトークンの Mint 方法
ここでやっと本題に入ります。上記を踏まえてプログラムを組んでいきます。
今回は ERC1155 でデプロイした Creator Contract のトークンを ERC1155LazyPayableClaim から Mint していきます。

```typescript:index.ts
import { ethers } from "ethers";
import abi from 'abi.json' // Etherscan から ERC1155LazyPayableClaim の ABI をJSON 形式でエクスポートし、ローカルに保存しておく

const ERC1155_LAZY_PAYABLE_CLAIM_ADDRESS = "0x44e94034AFcE2Dd3CD5Eb62528f239686Fc8f162"; // ERC1155LazyPayableClaim のコントラクトアドレス
const manifoldClaimIndex = 12345678; // クリエイターが initializeClaim のトランザクションを実行したときの引数に設定されていた引数 (etherscan から辿って控えておく)

const jsonRpcProvider = new ethers.providers.JsonRpcProvider("JSON-RPCエンドポイントのURL(Infura, Alchemy などから発行しておく)");
const wallet = new ethers.Wallet("トランザクションを実行するウォレットの秘密鍵", jsonRpcProvider);
const signer = wallet.connect(jsonRpcProvider);

const mintContract = new ethers.Contract(ERC1155_LAZY_PAYABLE_CLAIM_ADDRESS, JSON.stringify(abi), signer);

const functionArgs = [
  nftContractAddress, // Manifold でデプロイした Creator Contract 
  manifoldClaimIndex, 
  0, // mintIndex (Allow List 形式ではないので 0 で OK)
  [], // merkleProof (Allow List 形式 では無いので空配列で OK)
  wallet.address
];
const mintTx = {
  to: 'トークンの送付先アドレス',
  value: ethers.utils.parseEther("0.0005"), // 0.0005 ETH必須
  data: mintContract.interface.encodeFunctionData("mint", functionArgs),
};
const mintTxResponse = await wallet.sendTransaction(mintTx);
console.log(mintTxResponse);
```

上記ファイルを実行することで、トランザクションをNode.jsから実行することができます

## 注意点
**claimIndex がプログラムから直接取得することができない**、この一点に限ります。
Lazy Claim は Creator Contract に対して1個ずつデプロイされるわけではないので、おそらく Manifold 側の何かしらの識別子なのかと思われます。
自分が調べ尽くした限りではこの claimIndex という数値を取得する方法が見当たりませんでした...(知ってるよ！という方いらっしゃいましたらぜひコメントいただけると幸いです🙇‍♂️)


## まとめ
- Manifold の Creator Contract は ERC721, ERC1155 を拡張したアップグレード可能なスマートコントラクトで、コレクターが直接 NFT を Mint する機能は無い。
- Claim Page は Manifold の拡張機能(Apps)で、コレクターは Mint できるようにする。
- LazyClaimは Claim Page で実行するスマートコントラクトで、ERC721, ERC1155 ごとに別々に用意されており、それぞれ ERC721LazyPayableClaim, ERC1155LazyPayableClaim として定義されている。
- Claim PageでMintするには、 Creator Contract と Lazy Claim 間の紐付けが必要であり、紐付けは Manifold Studio の Apps ページから行う。
- LazyClaimに送信するMintトランザクションは、Manifold Claim Pageから実行する場合と同じ手順で実行できる。
- mint の引数である Claim Index の数値はプログラムから直接取得することができないため、手動で入力する必要がある。

## 余談
Claim App は手数料としてトランザクション送信者から 0.0005 ETH 請求していたのが個人的に意外でしたｗ
