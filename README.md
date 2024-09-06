# how2use_jaBART
京都大学が提供している日本語BARTが実装できない人のための空間
# 目次

- [初めに](#初めに)
- [requirement](#requirement)
- [その他](#その他)

# 初めに
京都大学が提供している日本語BARTが実装できない人のための空間

ファイルについては手順を明確化するためにpythonのjupyterを利用して作成

前提としてcondaが利用できることを要求

# requirement

###  今回利用するconda環境の作成

pythonのver.3.8を利用する
```
conda create -n [name] python=3.8
```
作成した仮想環境の実行
```
conda activate [name]
```
作成した環境の中でjupyterに対応できるように以下のコマンドをターミナルにて実施
```
conda install jupyter
```

### その他pipでインストールするものシリーズ
zenhan, pyknp, sentencepiece, tensorboardX(optional)
```
pip install zenhan pyknp sentencepiece tensorboardX
```

### 京大日本語BARTを使うためのfairseqの準備
[url](https://github.com/utanaka2000/fairseq)からcloneし、editableでinstall
```
git clone https://github.com/utanaka2000/fairseq
pip install --editable fairseq
```

### jumanpp install
jumanpp-2.0.0-rc3をwgetで取得してtarファイルを解凍
```
wget https://github.com/ku-nlp/jumanpp/releases/download/v2.0.0-rc3/jumanpp-2.0.0-rc3.tar.xz
tar xvf jumanpp-2.0.0-rc3.tar.xz
```
これにより得られたjumanpp-2.0.0-rc3ディレクトリから訂正する箇所
- jumanpp-2.0.0-rc3/libs/catch.hpp <6609>
-> ```char FatalConditionHandler::altStackMem[32768] = {};```に変更
- jumanpp-2.0.0-rc3/src/util/serialization_test.cc  <top>
-> ```#include <limits>```を追加

buildしていく。'where'のところの名前は任意でOK
```
mkdir jumanpp-2.0.0-rc3/bld
cd jumanpp-2.0.0-rc3/bld
cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX='where'
make install -j 4
```

### 前処理のコード準備
パスがややこしいので直接取ってくることにした。

[ここ](https://github.com/utanaka2000/fairseq/blob/japanese_bart_pretrained_model/jaBART_preprocess.py )からjaBART_preprocess.py持ってくる。

jaBART_preprocess.pyを開いて、<20>をちょっと訂正、jumanppのパスを指定しておく。

-> ```jumanpp = Juman(command = 'jumanpp-2.0.0-rc3/bld/where/bin/jumanpp')```

### BARTの準備
BART large v2.0 (3.7G)の場合
```
wget http://lotus.kuee.kyoto-u.ac.jp/nl-resource/JapaneseBARTPretrainedModel/japanese_bart_large_2.0.tar.gz
tar -zxvf japanese_bart_large_2.0.tar.gz
```
# その他
fairseq/fairseq/data/indexed_dataset.py に存在する```np.float```が使えない時は```float```に修正する

自分は<98>と<301>で訂正した

また、finetuneを実行していく際の、```--restore-file```があると思うが、```content/japanese_bart_large_2.0/bart_model.pt```のように```content```というのが必要

- finetune
```
fairseq-train dataset --arch bart_large --restore-file content/japanese_bart_large_2.0/bart_model.pt \
    --save-dir large2_model_save --tensorboard-logdir large2_tensorboard \
    --task translation_from_pretrained_bart --source-lang src --target-lang tgt \
    --criterion label_smoothed_cross_entropy --label-smoothing 0.2 --dataset-impl raw \
    --optimizer adam --adam-eps 1e-06 --adam-betas '{0.9, 0.98}' --lr-scheduler polynomial_decay --lr 5e-05 --min-lr -1 \
    --warmup-updates 2500 --total-num-update 10000 --weight-decay 0.3 \
    --max-tokens 1024 --update-freq 2 --save-interval -1 --no-epoch-checkpoints --log-format simple --log-interval 2 \
    --reset-optimizer --reset-meters --reset-dataloader --reset-lr-scheduler  --save-interval-updates 5000 \
    --ddp-backend no_c10d --max-update 20000 \
    --encoder-normalize-before --decoder-normalize-before --langs ja --prepend-bos --patience 5
```
- generate
```
fairseq-generate dataset_dir --path large2_model_save/checkpoint_best.pt --task translation_from_pretrained_bart \
    --dataset-impl raw --gen-subset test -s src -t tgt --max-sentences 64 --langs ja --prepend-bos > large2_ja2ja
```

- 後処理
pred file
```
grep ^H large2_ja2ja | LC_ALL=C sort -V | cut -f 3- | sed 's/<<unk>>/<unk>/g' | sed 's/▁//g' > large2_ja2ja.pred
```
src file
```
grep ^S large2_ja2ja | LC_ALL=C sort -V | cut -f 2- | sed 's/<<unk>>/<unk>/g' | sed 's/▁//g' > large2_ja2ja.src
```
tgt file
```
grep ^T large2_ja2ja | LC_ALL=C sort -V | cut -f 2- | sed 's/<<unk>>/<unk>/g' | sed 's/▁//g' > large2_ja2ja.tgt
```

