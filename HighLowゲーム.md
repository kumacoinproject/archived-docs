HighLowゲーム
====
コントラクト例として動くゲームを作りました。
大昔に流行ったSatoshiDiceのDappみたいなもの。
分散的に処理する事により不公平感を無くすことができる(?)。

ゲームの流れ
----
1. ユーザーの選択（High/Low）を調べる。
2. StartTXの挿入されたBlockhashをIntに変換。
3. Intを256で割った余りが128を境にHighかLowか調べる。
4. 勝ちならば、入金額の半分が加算されて送り返す。
5. 負けならば、入金額の半分が減算されて送り返される。
6. Storageに結果を加算する。

コントラクトの注意点
----
* 実用上必要な例外処理は考えていない（コードを簡単にする為。
* 使用するFnc名が今後変わる可能性があるが記事に反映しないかも。
* コントラクトが実行されなければ入金額そのままGOX。
* ContractInitでは、初期値として`{"win": 0, "lose": 0}`が必要。Keyチェックはせず。
* オーナーがコントラクトに残高を入れておかなければならない、ある程度。
* デバッグでいろいろな事をしているので数値がおかしかったりします。

コントラクトのソース
----
2018/12/26 ver0.0.18-alpha
```python=
from bc4py.contract.basiclib import *
from bc4py.contract.serializer import ContractTemplate


class HighLowGame(ContractTemplate):
    # ContractTemplateより受け継ぐ変数
    # start_tx        BLock   : StartTXのBlockオブジェクト
    # c_address       str     : コントラクトアドレス
    # c_storage       Storage : コントラクトストレージ
    # redeem_address  str     : お釣り用アドレス（償還アドレス）
    
    def __init__(self, *args):
        super().__init__(*args)

    def game(self, *args):
        # ユーザーの選択をBoolで取得
        f_high = bool(args[0])
        #　Storageの初期値を取得
        c_original = self.c_storage.copy()
        
        # StartTXの格納されたBlockオブジェクトを取得し、
        # Blockhashの数値に変換し256で割った余りを求める
        block = get_block_obj(height=self.start_tx.height)
        hash_int = int.from_bytes(block.hash, 'big') % 256
        
        # 償還する残高をAccountingオブジェクトで取得 => <Accounting {... 100000000}>}>
        # 償還する残高はStartTXによりコントラクトに入金されたコインを同額である
        returns = calc_return_balance(self.start_tx, self.c_address, self.redeem_address)
        
        # 賭け金は入金額に半分でid=0のコイン
        win_amount = returns[self.redeem_address][0] // 2
        # 勝敗を判定する
        if (f_high and 128 < hash_int) or (not f_high and 128 >= hash_int):
            # Win!
            self.c_storage['win'] += 1  # 勝ち数を１つ増やし、
            returns[self.redeem_address][0] += win_amount  # 償還残高を増やす
        else:
            # Lose..
            self.c_storage['lose'] += 1  # 負け数を１増やし、
            returns[self.redeem_address][0] -= win_amount  # 償還残高を減らす
        
        # Storageの差分を取得する
        c_diff = self.c_storage.export_diff(c_original)
        # 返り値は、（償還残高のAccountingオブジェクト、Storage差分）
        # ただし、消費したGasはユーザーに負担をしてもらうのが通例なので
        # 実際に返される額は少し少ない
        return returns, c_diff
```
以下より注意点を理解した上でContractをInitをしたものとする。
2018/12/25 テストネット上では**CLBKXHOTXTLK3FENVTCH6YPM5MFZS4BNAXFYNWBD**にデプロイ済み。

使い方
----
c_argsの第一引数により、True＝Hight、False＝Lowに賭ける事になる。

```bash
# Highを選択
curl --basic -u user:password "127.0.0.1:3000/private/contracttransfer" -H "Accept: application/json" -H "Content-Type: application/json" -d "{\"c_address\": \"CLBKXHOTXTLK3FENVTCH6YPM5MFZS4BNAXFYNWBD\", \"c_method\": \"game\", \"c_args\": [true]}"
# Lowを選択
curl --basic -u user:password "127.0.0.1:3000/private/contracttransfer" -H "Accept: application/json" -H "Content-Type: application/json" -d "{\"c_address\": \"CLBKXHOTXTLK3FENVTCH6YPM5MFZS4BNAXFYNWBD\", \"c_method\": \"game\", \"c_args\": [false]}"
```

ユーザーはStartTXがBlockに取り込まれたら数秒で結果が出るはず、
実行機の調子が悪いと実行されずGOXします、要Fix案件なのはわかってます。
実行が正しく反映されると [APIに表示されます](http://127.0.0.1:3000/public/getcontracthistory?c_address=CLBKXHOTXTLK3FENVTCH6YPM5MFZS4BNAXFYNWBD)。

実行例
----
StartTX、1.0コインをコントラクトアドレスに投入した。
```json=
{
    "hash": "f291712494df58f116ed7b87cebc5c2260fe646d8dc0b864b104deeceb91e460",
    "pos_amount": null,
    "height": 21479,
    "version": 2,
    "type": "TRANSFER",
    "time": 1544786910,
    "deadline": 1544797710,
    "inputs": [
        ["ca964236e1c79d29fb09756d5de0497eee055f2b048d8dfad88dcf3461f66551", 0]
    ],
    "outputs": [
        ["CLBKXHOTXTLK3FENVTCH6YPM5MFZS4BNAXFYNWBD", 0, 100000000],
        ["NBGHI55LBJWFK4FDLSHYNMQKBBH2WVPRSZ4ABU7R", 0, 55403318901]
    ],
    "gas_price": 100,
    "gas_amount": 10276,
    "message_type": "BYTE",
    "message": "01020901040528434c424b58484f5458544c4b3346454e565443483659504d354d465a5334424e415846594e574244050467616d6505284e43555343544b46334143564c4b574f5a46584a4b34374a365859545a46554a574c4849514855460901010001",
    "signature": [
        [
            "4353ad5c990d555a6770097971697e07b22387aa9ad3ba8e85bd9781bee97770", "6bfd3f795fe567b1f89e2af2dc55fb2a5efe2279f00b749e9102b57197a18117c4ab0dd2542a04bffbe92140f2ab0822f32eb5d775eb246e16b0c49b23407709"
        ]
    ],
    "f_on_memory": false,
    "size": 276,
    "total_size": 372
}
```

ConcludeTX、結果として勝ったようだ。1.479497コイン返ってきた。
0.020503コインの内、0.020269はTX作成に取られ、0.000234はGasとして使われた。
```json=
{
    "hash": "f99a3e4cdcd0cdfe5dfbad67e7e57f6f2d2ab93ed434f7fe7f8cd3e9f0845ebe",
    "pos_amount": null,
    "height": 21562,
    "version": 2,
    "type": "CONCLUDE_CONTRACT",
    "time": 1544786910,
    "deadline": 1544797710,
    "inputs": [
        ["5c3ef68f52b36016a6236894a480ca5bc6c75ea4245943b1790a6b7006421ce5", 1]
    ],
    "outputs": [
        ["NCUSCTKF3ACVLKWOZFXJK47J6XYTZFUJWLHIQHUF", 0, 147949700],
        ["CLBKXHOTXTLK3FENVTCH6YPM5MFZS4BNAXFYNWBD", 0, 96492955000]
    ],
    "gas_price": 100,
    "gas_amount": 20269,
    "message_type": "BYTE",
    "message": "01020901030528434c424b58484f5458544c4b3346454e565443483659504d354d465a5334424e415846594e574244070120f291712494df58f116ed7b87cebc5c2260fe646d8dc0b864b104deeceb91e4600b0101050377696e010103",
    "signature": [
        ["1a9a0a3194892622ef6fee5acb66b5e29f1d2ebbd9c9ed00c3eef2e4b8c53ca3", "40c5ee5b730b6c34b08ada181d33ae2ad9d56c1ec364b657ff18322be99d6598375d5c5a10bc0849b767daba9f5b51b864a723dce6e36e41e322641c739c3f0a"],
        ["5cb6a3d317ea26169459d645c74da4bc060f6f153ee352a405de9cda40318aa4", "726512ea5317f50763c1e3946b31ac4268946575001b13439fa1345ca415c732da6ea5cb257b1501e61fd423e2ecc343dbd1cc1d7f96a718c2f79708673d7e04"]
    ],
    "f_on_memory": false,
    "size": 269,
    "total_size": 461
}
```

結果
----
単純な使用例のみの現状からある程度動くコントラクトを構築出来てホッと一息。
良い点として、Python使いなら理解しやすそうなコントラクト構成となった気がする。
問題点として、~~コントラクト実行機に不安定な様子がみられ、一度失敗するとリトライ処理は
ないのでそのままになる。~~ また、**basiclib**が充実からほど遠いので何が必要になるか
見極める事が課題になりそう。

2018/11/10