PyContractについて
====
コントラクトの紹介、操作を一通り示す暫定版

特徴
----
### BlockChain
* UTXO型（Bitcoinと同じ、EthはAccount型、CPもAccount型、NEMもAccount型
* Mining consensusはPoW、PoS、単独、複数、自由に選べるが未定
* CoinにColorを付ける事ができる、なので鋳造(Mintcoin)と呼ぶ

### PythonのコードがContractとしてそのまま動く
* PythonVertualMachine（PVM）をコントラクト実行機として使用
* 互換性・一般性を考えるとSolidityだがPython好きなのでPythonで
* Pythonを用いる利点
    * 抽象度が高いためにコードを組みやすい
    * 新規に言語を憶える必要がない
* 問題点
    * Objectを直列化するライブラリ”Pickle”の危険性 [参考](https://qiita.com/tanuk1647/items/4c2a305c7cc4e12ef99d)
    * ユーザーがContractのコードレビューをしなければならない
    * Pythonの仕様変更によりバグる可能性あり
* Pickleの危険性を全てつぶしたはず
* 外部ライブラリをImportしなければ手ぶらで安全と言えるようにしたい

### CoreもPythonで書かれている
* ボトルネックはDBなので問題になりにくい（LevelDB、Sqlite
* BTCやETH以外で問題になるほど使われているのか疑問
* Pythonでないと開発リソースの絶対量が足りない
* 暗号関係ライブラリを自身の管理するNEMより流用できる
* MultiProcessingで速度を上げているので多コア有利
* もしダメそうならCythonという魔術も考えるかもしれない

### 結果含有型コントラクト
* Etheriumと異なる仕組み・思想を持つ
* Ethの場合
    * 1.Contractの実行計画がチェーンに書き込まれる
    * 2.EVMが実行され内部のアカウント情報に反映
    * いつ実行しても”結果は常に同じ”となる
    * ⇒ チェーン外部の状態を反映できない
* PyContractの場合
    * 1.Contractの実行計画がチェーンに書き込まれる
    * 2.PVMが実行され結果を反映したTXが生成される
    * 3.Validatorが署名しチェーンに書き込まれる
    * 結果は”チェーンに含有”される
    * ⇒ チェーン外部の状態を反映できる
* PyContractのようなものを [オラクル・システム](http://block-chain.jp/blockchain/oracle-blockchain/)という
* 問題点
    * 他のValidatorと結果のすり合わせを行うのがシビア
    * 内部ストレージをダイナミックに変化できない
    * 外部の状態を反映するのでコントラクト不成立になりやすい

### BlockChainとContractを分離している
* 原因は拡張性と利便性と安全性を求めた事による
* 拡張性とは
    * ”任意のライブラリをImportできる”こと
    * Contractを完了までの制限時間・実行コストの撤廃
    * Contractで何でも動く、GPUだろうが、AIだろうが
* 利便性とは
    * 一般ユーザーがNodeを設置する必要が無い（はず
    * ChainがContractの実行コストを支払う必要がない
    * ContractコードをUpdateする事ができる
* 安全性とは
    * Contractの漸弱性からチェーンを分離できる
    * Contractがチェーン外の状態に左右されやすい
    * 不安定なContractは不成立として処理できる
* Contractは一般ユーザーが設置する非運営のSuperNode制
* ただし、拡張性を使用しなければ安全にContractを実行できる（はず
* 分離しているのでCoreのみをCで書き直すことも可能（意味ない


基本コントラクト
----
以下を前提とする
* 既にValidatorアドレスを作成済み
* Contractアドレスに実行用のコインを送金済み
* 具体的なコマンドはAPI-Doc参照
```python
class Contract:
    def __init__(self, *args):
        # 引数の増加に対応する為
        self.start_tx = args[0]
        self.c_address = args[1]
        self.c_storage = args[2]
        self.redeem_address = args[3]

    def update(self, *args):
        # 基本操作なしで承認
        # c_bin, c_extra_imports, c_settings = args
        return None, None

    def store_coins(self, *args):
        returns = calc_return_balance(self.start_tx, self.c_address, self.redeem_address)
        returns[self.redeem_address][0] -= 10000  # 返金額から0.0001引く=コントラクトに蓄える
        return returns, None

    def edit_storage(self, *args):
        key, value, *dummy = args
        storage_original = self.c_storage.copy()  # 元のStorageを保存
        self.c_storage[key] = value
        c_diff = self.c_storage.export_diff(storage_original)  # 元のStorageと比較出力
        returns = calc_return_balance(self.start_tx, self.c_address, self.redeem_address)
        return returns, c_diff
```
Class型。Compile時に名前で判断する為、class名は必ず`Contract`でなければならない。

Contractのバイナリ
----
`/private/sourcecompile`を用いてコンパイルすると生成される。
上記のコードで982byte生成される。内容は`pickletools`で調べる事ができる。Pythonそのままだから性質としてはｽｸﾘﾌﾟﾄ言語。
```
800495cb030000000000008c1962633470792e636f6e74726163742e64696c6c2e5f64696c6c948c0c5f6372656174655f747970659493942868008c0a5f6c6f61645f747970659493948c047479706594859452948c08436f6e74726163749468048c066f626a656374948594529485947d94288
c0a5f5f6d6f64756c655f5f948c085f5f6d61696e5f5f948c075f5f646f635f5f944e8c085f5f696e69745f5f9468008c105f6372656174655f66756e6374696f6e9493942868048c08436f6465547970659485945294284b014b004b024b024b47432c7c01640119007c005f007c01640219007c005f017c0164
0319007c005f027c01640419007c005f036400530094284e4b004b014b024b037494288c0873746172745f7478948c09635f61646472657373948c09635f73746f72616765948c0e72656465656d5f616464726573739474948c0473656c66948c0461726773948694680868114b05430800010a010a010a01942
929749452946362633470792e636f6e74726163742e746f6f6c730a5f5f646963745f5f0a68114e4e7d94749452948c06757064617465946813286816284b014b004b024b024b47430464015300944e4e4e8694869429681e681f8694680868274b0b43020002942929749452946362633470792e636f6e747261
63742e746f6f6c730a5f5f646963745f5f0a68274e4e7d94749452948c0b73746f72655f636f696e73946813286816284b014b004b034b044b47433074007c006a017c006a027c006a0383037d027c027c006a0319006401050019006402380003003c007c02640066025300944e4b004d10278794288c1363616
c635f72657475726e5f62616c616e6365946819681a681c7494681e681f8c0772657475726e73948794680868324b0f4306000112011601942929749452946362633470792e636f6e74726163742e746f6f6c730a5f5f646963745f5f0a68324e4e7d94749452948c0c656469745f73746f726167659468132868
16284b014b004b084b044b4743447c015e027d027d037d047c006a006a0183007d057c037c006a007c023c007c006a006a027c0583017d0674037c006a047c006a057c006a0683037d077c077c0666025300944e859428681b8c04636f7079948c0b6578706f72745f646966669468356819681a681c749428681
e681f8c036b6579948c0576616c7565948c0564756d6d79948c1073746f726167655f6f726967696e616c948c06635f6469666694683774946808683f4b14430c00010a010a010a010c011201942929749452946362633470792e636f6e74726163742e746f6f6c730a5f5f646963745f5f0a683f4e4e7d947494
529475749452942e
```

Chainに書き込む
----
`/private/contractinit`を用いてBlockChainに書き込む。この時点ではチェーンにデータが"載っている"だけであり効力はなし。このTXの事を**start_tx**と呼び、この状態を”アンカー”されていると呼んでいる。チェーンに投錨してTXを固定したイメージ。
このTXのFeeはTX作成者が支払い、またコントラクト実行にかかるGasも余分に支払っておく事になる。

次にマニュアルで`/private/concludecontract`によりコントラクトの結果を書き込む。このTXの事を**conclude_tx**と呼ぶ。**start_hash**によりstart_txを指定し、**storage**で内部ストレージの初期状態を指定できる。チェーンから見ると人間がコントラクトを実行して結果を書き込んでいるように見え、まだコントラクト自体が存在しないのでマニュアルで書き込まなければならない。自動化するか未定。

チェーンに取り込まれたのを確認したら`/public/getcontractinfo`で反映されている事を確認する。


コントラクトを実行してみる　(Return coins)
----
`/private/contracttransfer`によりstart_txを書き込む。
```bash=
curl --basic -u user:password "127.0.0.1:3003/private/contracttransfer" -H "Accept: application/json" -H "Content-Type: application/json" -d "{\"c_address\": \"CJ4QZ7FDEH5J7B2O3OLPASBHAFEDP6I7UKI2YMKF\", \"c_method\": \"store_coins\", \"c_args\": []}"
```
methodの **store_coins** を実行してみる。内容としてはContractに0.001蓄えて残りをユーザーに返すというもので基本的な機能を示す。

この様なStartTXが生成される。
```json=
{
    "hash": "f6d26de7a9af547473de6b04ef7baf3e501b8a5741ccaa08102025c586d1c079",
    "pos_amount": null,
    "height": 901,
    "version": 2,
    "type": "TRANSFER",
    "time": 1544198028,
    "deadline": 1544208828,
    "inputs": [
        ["21459760b3913cb8091dd3a9767025ccf07d476bad70ee96b2acad180eeb5ee4", 0]
    ],
    "outputs": [
        ["CJ4QZ7FDEH5J7B2O3OLPASBHAFEDP6I7UKI2YMKF", 0, 100000000],
        ["NAZU3RLJDJGYHRBCU57XE4TL6LOU5UJLK7YBDEMO", 0, 55171953033]
    ],
    "gas_price": 100,
    "gas_amount": 10281,
    "message_type": "BYTE",
    "message": "01020901040528434a34515a3746444548354a3742324f334f4c50415342484146454450364937554b4932594d4b46050b73746f72655f636f696e7305284e4157344b48484d4845584e3334364a504152564533543354423655593556564743575033514849090100",
    "signature": [
        [
            "e2a4b24a7ad26a9f305e28d79613ede9610d9d39c8f23318fa1325fcb21a2404",
            "09ce5cafff132d4b983191124e7ac132a76ff7a0a5def859d03d8219bd887cc709ffd4c171c3305371780b9248315431d02c5f1b6422efb15bbda2a78c0c2903"
        ]
    ],
    "f_on_memory": true,
    "size": 281,
    "total_size": 377
}
```

EmulatorがStartTXが取り込まれるのを検知すると、ConcludeTXを生成する。
```json=
{
    "hash": "45ee8866368c46d894b8de66336cf20c2a20d40040770b2d48938db82815ebee",
    "pos_amount": null,
    "height": 903,
    "version": 2,
    "type": "CONCLUDE_CONTRACT",
    "time": 1544198028,
    "deadline": 1544208828,
    "inputs": [
        ["6c9faaf7374606ba08ead6b5a2b49489419cebf65f421247d3a372f3642a166a", 0]
    ],
    "outputs": [
        ["NAW4KHHMHEXN346JPARVE3T3TB6UY5VVGCWP3QHI", 0, 97962000],
        ["CJ4QZ7FDEH5J7B2O3OLPASBHAFEDP6I7UKI2YMKF", 0, 97896947600]
    ],
    "gas_price": 100,
    "gas_amount": 20259,
    "message_type": "BYTE",
    "message": "01020901030528434a34515a3746444548354a3742324f334f4c50415342484146454450364937554b4932594d4b46070120f6d26de7a9af547473de6b04ef7baf3e501b8a5741ccaa08102025c586d1c0790c",
    "signature": [
        [
            "c39e698bd177cb020e6723f797b3d469bc96fde0883c000d1fd40ff99df93173",
            "964eb7afb92e0ad5a9c8bef8c80e79058928a5eae0bdbb1aff39240506316ae8e7449f5bb9df744f8a0bbb55316adc538c19a22faede88a25958298fdd711a01"
        ],
        [
            "8975a4dda4a95dd2400aa186b2b0fb2bbf80bc1981df7c03bd97f957c57ccc87",
            "6f8389b2426430da00184c2a63a3097326b0976ca2aa316f1a47e949ec917a6b417483afd4c1690fdda4d6b733d5cd214fe4db0c43a3c73571e8ffe305236205"
        ]
    ],
    "f_on_memory": true,
    "size": 259,
    "total_size": 451
}
```
Emulatorのログを確認。コントラクトアドレスに100000000入金され99990000償還されている事がわかる。実際のConcludeTXを確認すると97962000(-20280*100)まで減っている。これは使用したEmulateGas(21)とTxFee(20259)を引かれたからである。
```javascript=
[DEBUG ] [Emulator  ]  wait for notify of listen port.
pdb is running on 127.0.0.1:52346
[DEBUG ] [Emulator  ]  Communication port=52346.
[DEBUG ] [Emulator  ]  Start emulation of <TX 901 TRANSFER f6d26de7a9af547473de6b04ef7baf3e501b8a5741ccaa08102025c586d1c079>
[DEBUG ] [Emulator  ]  Finish 2.941Sec error:"None"
[INFO  ] [Emulator  ]  Success gas=21 line=28 result=(<Account {'NAW4KHHMHEXN346JPARVE3T3TB6UY5VVGCWP3QHI': <Balance {0: 99990000}>}>, None)
[DEBUG ] [Emulator  ]  Close file obj 3282903584344.
[DEBUG ] [Emulator  ]  Retry calculate tx fee. [20174=>259+20000=20259]
[DEBUG ] [Emulator  ]  Move conclude fee 0:2028000
[DEBUG ] [Emulator  ]  Verify signature 1tx
[DEBUG ] [Emulator  ]  Check unconfirmed tx 45ee8866368c46d894b8de66336cf20c2a20d40040770b2d48938db82815ebee
[INFO  ] [Emulator  ]  Marge contract tx <TX None CONCLUDE_CONTRACT 45ee8866368c46d894b8de66336cf20c2a20d40040770b2d48938db82815ebee>
[INFO  ] [Emulator  ]  Broadcast success <TX None CONCLUDE_CONTRACT 45ee8866368c46d894b8de66336cf20c2a20d40040770b2d48938db82815ebee>
```

`/public/getcontracthistory`でコントラクトの実行履歴を確認すると以下の様に出る。
```json=
{
    "index": 1,
    "height": 903,
    "start_hash": "f6d26de7a9af547473de6b04ef7baf3e501b8a5741ccaa08102025c586d1c079",
    "finish_hash": "45ee8866368c46d894b8de66336cf20c2a20d40040770b2d48938db82815ebee",
    "c_method": "store_coins",
    "c_args": [],
    "c_storage": null,
    "redeem_address": "NAW4KHHMHEXN346JPARVE3T3TB6UY5VVGCWP3QHI"
}
```

コントラクトを実行してみる　(Store storage)
----
```bash=
curl --basic -u user:password "127.0.0.1:3000/private/contracttransfer" -H "Accept: application/json" -H "Content-Type: application/json" -d "{\"c_address\": \"CJ4QZ7FDEH5J7B2O3OLPASBHAFEDP6I7UKI2YMKF\", \"c_method\": \"edit_storage\", \"c_args\": [\"name\", \"namuyan\"]}"
```
method`edit_storage`を使用してStorageに`"name"=>"namuyan"`を書きこんでみます。


```javascript=
[DEBUG ] [Emulator  ]  Communication port=53206.
[DEBUG ] [Emulator  ]  Start emulation of <TX 1191 TRANSFER 145c9213f35960525e09ef70c025f5a68ba0453ff8cd6a379db4fad91da68ab6>
[DEBUG ] [Emulator  ]  Finish 2.88Sec error:"None"
[INFO  ] [Emulator  ]  Success gas=24 line=49 result=(<Accounting {'NA6DENGWZQ35WA6BVRZF5LCJVUK5DJEX5VNUYKS2': <Balance {0: 100000000}>}>, {'name': 'namuyan'})
[DEBUG ] [Emulator  ]  Close file obj 2138022308744.
[DEBUG ] [Emulator  ]  Retry calculate tx fee. [20191=>309+20000=20309]
[DEBUG ] [Emulator  ]  Move conclude fee 0:2033300
[DEBUG ] [Emulator  ]  Verify signature 1tx
[DEBUG ] [Emulator  ]  Check unconfirmed tx 8e7f5b6785408740d6cd8c81fd31031338ef7261434617c3610b497810ab44a6
[INFO  ] [Emulator  ]  Success broadcast new tx <TX None CONCLUDE_CONTRACT 8e7f5b6785408740d6cd8c81fd31031338ef7261434617c3610b497810ab44a6>
[INFO  ] [Emulator  ]  Broadcast success <TX None CONCLUDE_CONTRACT 8e7f5b6785408740d6cd8c81fd31031338ef7261434617c3610b497810ab44a6>
```
成功するとこのようなログが確認されます。Emulationに時間がかかっているのは、PVMを分離する為にSocketServerを立ち上げに時間がかかっている為です。

#### StartTX
```json=
{
    "hash": "145c9213f35960525e09ef70c025f5a68ba0453ff8cd6a379db4fad91da68ab6",
    "pos_amount": null,
    "height": 1191,
    "version": 2,
    "type": "TRANSFER",
    "time": 1544231955,
    "deadline": 1544242755,
    "inputs": [
        ["347ac40430941309bd742d538d833733c7c30858f37969b8ba16296e7e436876", 0]
    ],
    "outputs": [
        ["CJ4QZ7FDEH5J7B2O3OLPASBHAFEDP6I7UKI2YMKF", 0, 100000000],
        ["NB5CL3VY3RYQZNP33PYOKITFBH5F7S5DEVXVOQUG", 0, 55178349685]
    ],
    "gas_price": 100,
    "gas_amount": 10297,
    "message_type": "BYTE",
    "message": "01020901040528434a34515a3746444548354a3742324f334f4c50415342484146454450364937554b4932594d4b46050c656469745f73746f7261676505284e413644454e47575a5133355741364256525a46354c434a56554b35444a455835564e55594b533209010205046e616d6505076e616d7579616e",
    "signature": [
        [
            "563d9f6f4c5818a8b1f5837250b3169cde9a41ef5482f42ffaeccbdabd3e62d0",
            "fb451490f5b69290894730f2ea981be67f7d25bfd0977a5efe20b86aa508c42a913c8afb6de835b9e2f890d45010b7641195dc368a304fff581c9aaffa31b80d"
        ]
    ],
    "f_on_memory": true,
    "size": 297,
    "total_size": 393
}
```

#### CocludeTX
```json=
{
    "hash": "8e7f5b6785408740d6cd8c81fd31031338ef7261434617c3610b497810ab44a6",
    "pos_amount": null,
    "height": null,
    "version": 2,
    "type": "CONCLUDE_CONTRACT",
    "time": 1544231955,
    "deadline": 1544242755,
    "inputs": [
        ["20ebeb4e42792406e631d4b9e41775c3a17551dc40ed6be2dd3dfe767f018174", 0],
        ["45ee8866368c46d894b8de66336cf20c2a20d40040770b2d48938db82815ebee", 1]
    ],
    "outputs": [
        ["NA6DENGWZQ35WA6BVRZF5LCJVUK5DJEX5VNUYKS2", 0, 97966700],
        ["CJ4QZ7FDEH5J7B2O3OLPASBHAFEDP6I7UKI2YMKF", 0, 97896950000]
    ],
    "gas_price": 100,
    "gas_amount": 20309,
    "message_type": "BYTE",
    "message": "01020901030528434a34515a3746444548354a3742324f334f4c50415342484146454450364937554b4932594d4b46070120145c9213f35960525e09ef70c025f5a68ba0453ff8cd6a379db4fad91da68ab60b010105046e616d6505076e616d7579616e",
    "signature": [
        [
            "8975a4dda4a95dd2400aa186b2b0fb2bbf80bc1981df7c03bd97f957c57ccc87",
            "0a076eaef08e8242c78ee0b5a119a531f4fd675a92e4a5a46462a553ffca593c5419bb0a891b560d1166b63b3ca11304bd894c811771c27ff64340f6fd7a3607"
        ]
    ],
    "f_on_memory": true,
    "size": 309,
    "total_size": 405
}
```

`/public/getcontracthistory`でコントラクトの実行履歴を確認すると以下の様に出る。
```json=
{
    "index": 2,
    "height": 1519,
    "start_hash": "145c9213f35960525e09ef70c025f5a68ba0453ff8cd6a379db4fad91da68ab6",
    "finish_hash": "8e7f5b6785408740d6cd8c81fd31031338ef7261434617c3610b497810ab44a6",
    "c_method": "edit_storage",
    "c_args": [
        "name",
        "namuyan"
    ],
    "c_storage": {
        "name": "namuyan"
    },
    "redeem_address": "NA6DENGWZQ35WA6BVRZF5LCJVUK5DJEX5VNUYKS2"
}
```
