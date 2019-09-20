# asr-scratch
# Script preparation
To train the speech recognition from scratch you first need to compile Kaldi from the source (see the instructions in the INSTALL file from Kaldi source repository https://github.com/kaldi-asr/kaldi). Then go to `egs` folder within kaldi project that you have compiles and create your project folder. Also, you need to use scripts from `egs/wsj/s5` located in 2 folders `steps` and `utils`. You also need to create a `cmd.sh` and `path.sh` as well. Here are the commands:
```
cd egs
mkdir -p fotolia/s5
cd fotolia/s5
ln -s ../../wsj/s5/steps steps
ln -s ../../wsj/s5/utils utils
echo export train_cmd=run.pl >> cmd.sh
echo export decode_cmd=run.pl >> cmd.sh
chmod +x cmd.sh
echo export KALDI_ROOT=`pwd`/../../.. >> path.sh
echo -e \\n#SRILM >> path.sh
echo export SRILM=$KALDI_ROOT/tools/srilm >> path.sh
echo export PATH=$SRILM/bin/i686-m64:$SRILM/bin:$PWD/utils:$KALDI_ROOT/src/featbin:$KALDI_ROOT/tools/openfst-1.6.2/bin:$KALDI_ROOT/src/fstbin:$KALDI_ROOT/src/lmbin:$KALDI_ROOT/src/gmmbin:$KALDI_ROOT/src/bin:$KALDI_ROOT/src/latbin:$KALDI_ROOT/src/nnet2bin:$KALDI_ROOT/src/nnetbin:$KALDI_ROOT/src/online2bin:$KALDI_ROOT/src/ivectorbin:$KALDI_ROOT/src/nnet3bin:$PATH >> path.sh
echo LD_LIBRARY_PATH=$KALDI_ROOT/tools/liblbfgs-1.10/lib/.libs:$LD_LIBRARY_PATH >> path.sh
echo export MANPATH=$SRILM/man >> path.sh
echo -e \\n# Variable needed for proper data sorting >> path.sh
echo export LC_ALL=C >> path.sh
chmod +x path.sh
```

# Data preparation
## Audio data
Use Amazon Mechanical Turk to collect audio files as a training and test set if you don't have them. For example: `data/audio/train` and `data/audio/test` are two folders contain the collected .wav files from the Fotolia queries. Each file has the following format: TurkerID_QueryID_Word1_Word2_..._WordN.wav. 
## Prepare acoustic data

### Create spk2gender files
```
export LC_ALL=C
mkdir -p data/train data/test
cd data/train
ls ../../audio/train | cut -d'_' -f1|sed 's/$/_ m/'|sort|uniq > spk2gender
cd ../test
ls ../../audio/test | cut -d'_' -f1|sed 's/$/_ m/'|sort|uniq > spk2gender
```
The output are 2 `spk2gender` files, each for `train` and `test` respectively. Each line of `spk2gender` contains `speakerId gender` where gender is either `m` (male) or `f` (female). In the above script, I just assign all speakers' gender to male, but you should assign the correct gender if you know it.
### Create wav.scp files
```
export LC_ALL=C
cd ../train
ls ../../audio/train | cut -d'.' -f1 > wav1.scp
cd ../../audio/train
ls > ../../data/train/wav2.scp
sed -i 's,^,'$PWD'\/,' ../../data/train/wav2.scp
cd ../../data/train
paste -d' ' wav1.scp wav2.scp | sort > wav.scp
rm wav1.scp wav2.scp
cd ../test
ls -1 ../../audio/test | cut -d'.' -f1 > wav1.scp
cd ../../audio/test
ls > ../../data/test/wav2.scp
sed -i 's,^,'$PWD'\/,' ../../data/test/wav2.scp
cd ../../data/test
paste -d' ' wav1.scp wav2.scp | sort > wav.scp
rm wav1.scp wav2.scp
```
Each line of the wav.scp file contains `utteranceId fullPathToUtteranceAudioFile`.
### Create text, spk2utt, utt2spk files
```
export LC_ALL=C
cd ../train/
cut -d' ' -f1 wav.scp > spk_utt_
cut -d'_' -f3- spk_utt_ |sed 's/_/ /g' > utt
paste -d' ' spk_utt_ utt > text
rm utt
cd ../test/
cut -d' ' -f1 wav.scp > spk_utt_
cut -d"_" -f3- spk_utt_ |sed 's/_/ /g' > utt
paste -d' ' spk_utt_ utt > text
rm utt
cd ../train
cut -d'_' -f1 spk_utt_ |sed 's/$/_/' > spk
paste -d' ' spk_utt_ spk > utt2spk
../../utils/utt2spk_to_spk2utt.pl utt2spk > spk2utt
rm spk_utt_ spk
cd ../test
cut -d'_' -f1 spk_utt_ |sed 's/$/_/' > spk
paste -d" " spk_utt_ spk > utt2spk
../../utils/utt2spk_to_spk2utt utt2spk.pl > spk2utt
rm spk_utt_ spk
cd ../../
```
Each line of file `text` contains `utteranceId word1 word2 wordN`. Each line of `spk2utt` contains `speakerId` and all utterance ids of that speaker. Each line of `utt2spk` contains `utteranceId speakerId`. `spk2utt` and `utt2spk` are in sorted order.
### Validate the data
```
utils/validate_data_dir.sh --no-feats data/train
utils/validate_data_dir.sh --no-feats data/test
```
### Create corpus.txt and vocab-full.txt files
```
export LC_ALL=C
cd data
mkdir local
cd local
cut -d' ' -f2- ../train/text > corpus.txt
cat corpus.txt | sed -r 's/[[:space:]]+/\n/g' | sed '/^$/d' | sort | uniq > vocab-full.txt
cd ../../
```
Basic language model: File `corpus.txt` contains all utterances in `train`. File `vocab-full.txt` contains all distinct words in `train` in the alphabetic order.
Extended language mode: You can add more sentences to the end of file `corpus.txt` and regenerate file `vocab-full.txt` to train an extended language model.
## Extract features
### Extract MFCC features from audio files
```
. ./path.sh || exit 1
. ./cmd.sh || exit 1
nj=88 #number of parallel jobs
mfccdir=mfcc
steps/make_mfcc.sh --nj $nj --cmd "$train_cmd" data/train exp/make_mfcc/train $mfccdir
steps/make_mfcc.sh --nj $nj --cmd "$train_cmd" data/test exp/make_mfcc/test $mfccdir
```
Output: 
* `raw_mfcc_xxx_n.scp` and `raw_mfcc_xxx_n.ark` files located in folder `mfcc`. Each line of file `.scp` contains `utteranceId fullPathToTheARKFile:n` where `n` is the [file position](http://www.gnu.org/software/libc/manual/html_node/File-Position.html). The `.ark` file contains features of utterances. Each utterance is stored in `.ark` as a matrix of `n x m` where `n` is the number of frames and `m` if the feature dimension. See [Make MFCCs](https://git.corp.adobe.com/bui/asr/wiki/Make-Mel-Frequency-Cepstral-Coefficient-(MFCC)-vectors) for the details.
* data/train/feats.scp and data/test/feats.scp

### Compute cepstral mean and variance statistics per speaker
```
steps/compute_cmvn_stats.sh data/train exp/make_mfcc/train $mfccdir
steps/compute_cmvn_stats.sh data/test exp/make_mfcc/test $mfccdir
```
The outputs are 4 files `cmvn_train.scp`, `cmvn_train.ark`, `cmvn_test.scp`, `cmvn_test.ark` located in folder `mfcc`
## Prepare language data
### Create lexicon.txt, nonsilence_phones.txt, optional_silence.txt, silence_phones.txt
`local/prepare_dict.sh`
### Prepare language data
```
utils/prepare_lang.sh data/local/dict "<UNK>" data/local/lang data/lang
```
## Create n-gram language model
### Create lm.arpa file
```
. ./path.sh || exit 1
local=data/local
mkdir $local/tmp
ngram-count -order $lm_order -write-vocab $local/tmp/vocab-full.txt -wbdiscount -text $local/corpus.txt -lm $local/tmp/lm.arpa
```
Output: File `lm.arpa`. See this [link](http://www.speech.sri.com/projects/srilm/manpages/ngram-format.5.html) if you are not familiar with the file format.
### Make G.fst file
```
lang=data/lang
arpa2fst --disambig-symbol=#0 --read-symbol-table=$lang/words.txt $local/tmp/lm.arpa $lang/G.fst
```
Output: File `G.fst` is a binary file, but you can use the command `../../../tools/openfst-1.6.2/bin/fstprint data/lang/G.fst` to read the content. It returns a 5-column matrix. Column 1 is ... Column 2 is .... Column 3 is .... Column 4 is .... Column 5 is ...
## Create GMM-HMM acoustic model
### Train monophones
```
nj=88
. ./cmd.sh
. ./path.sh
steps/train_mono.sh --nj $nj --cmd "$train_cmd" data/train data/lang exp/mono
```
* Input: Two folders: `data/train` and `data/lang`
* Output: Folder exp/mono. Files in this folder are ali.xx.gz, fsts.xx.gz
### Decode
```
utils/mkgraph.sh --mono data/lang exp/mono exp/mono/graph || exit 1
steps/decode.sh --config conf/decode.config --nj $nj --cmd "$decode_cmd" exp/mono/graph data/test exp/mono/decode
```
### Align
```
steps/align_si.sh --nj $nj --cmd "$train_cmd" data/train data/lang exp/mono exp/mono_ali || exit 1
```
* Input: Three folders data/train, data/lang, exp/mono
* Output: exp/mono_ali
### Train delta-based triphones from aligned monophones (tri1)
```
steps/train_deltas.sh --cmd "$train_cmd" 2000 10000 data/train data/lang exp/mono_ali exp/tri1 || exit 1
```
* Input: Three folders data/train, data/lang, and exp/mono_ali
* Output: Folder exp/tri1
### Decode 
```
utils/mkgraph.sh data/lang exp/tri1 exp/tri1/graph || exit 1
steps/decode.sh --config conf/decode.config --nj $nj --cmd "$decode_cmd" exp/tri1/graph data/test exp/tri1/decode
```
* Input: Two folders exp/tri1 and data/test
* Output: Folder exp/tri1/decode
### Align
```
steps/align_si.sh --nj $nj --cmd "$train_cmd" \
  --use-graphs true data/train data/lang exp/tri1 exp/tri1_ali || exit 1;
```
* Input: Folders data/train, data/lang, and exp/tri1
* Output: Folder exp/tri1_ali
### Train LDA+MLLT from delta-based triphones (tri2b)
```
steps/train_lda_mllt.sh --cmd "$train_cmd" 2500 15000 \
 data/train data/lang exp/tri1_ali exp/tri2b || exit 1;
```
* Input: Folders data/train, data/lang and exp/tr1_ali
* Output: Folder exp/tri2b
### Decode
```
utils/mkgraph.sh data/lang exp/tri2b exp/tri2b/graph
steps/decode.sh --config conf/decode.config --nj $nj --cmd "$decode_cmd" \
  exp/tri2b/graph data/test exp/tri2b/decode
```
* Input: Folders exp/tri2b, data/test
* Output: Folder exp/tri2b/decode
### Align
```
steps/align_si.sh --nj $nj --cmd "$train_cmd" --use-graphs true \
   data/train data/lang exp/tri2b exp/tri2b_ali || exit 1;
```
* Input: Folders data/train, data/lang/, and exp/tri2b
* Output: Folder exp/tri2b_ali
### Train MPE from aligned LDA+MLLT (tri2b_mpe)
```
steps/train_mpe.sh data/train data/lang exp/tri2b_ali exp/tri2b_denlats exp/tri2b_mpe || exit 1;
```
* Input: Folders data/train and data/lang
* Output: Folders exp/tri2b_denlats and exp/tri2b_mpe
### Decode
```
steps/decode.sh --config conf/decode.config --iter 4 --nj $nj --cmd "$decode_cmd" \
   exp/tri2b/graph data/test exp/tri2b_mpe/decode_it4 || exit 1;
```
* Input: Folders exp/tri2b/graph and data/test
* Output: Folder exp/tri2b_mpe/decode_it4
### Train SAT from aligned LDA+MLLT (tri3b)
```
steps/train_sat.sh 3000 20000 data/train data/lang exp/tri2b_ali exp/tri3b || exit 1;
```
* Input: Folders data/train, data/lang, and exp/tri2b_ali
* Output: Folder exp/tri3b
### Decode
```
utils/mkgraph.sh data/lang exp/tri3b exp/tri3b/graph || exit 1;
steps/decode_fmllr.sh --config conf/decode.config --nj $nj --cmd "$decode_cmd" \
  exp/tri3b/graph data/test exp/tri3b/decode || exit 1;
```
* Input: Folders exp/tri3b/graph and data/set
* Output: Folder exp/tri3b/decode
### Align all data with LDA+MLLT+SAT system (tri3b)
```
steps/align_fmllr.sh --nj $nj --cmd "$train_cmd" --use-graphs true \
  data/train data/lang exp/tri3b exp/tri3b_ali || exit 1;
```
* Input: Folders data/train, data/lang, and exp/tri3b
* Output: Folder exp/tri3b_ali
### Train MMI on top of tri3b (i.e. LDA+MLLT+SAT+MMI)
```
steps/make_denlats.sh --config conf/decode.config \
   --nj $nj --cmd "$train_cmd" --transform-dir exp/tri3b_ali \
  data/train data/lang exp/tri3b exp/tri3b_denlats || exit 1;
steps/train_mmi.sh data/train data/lang exp/tri3b_ali exp/tri3b_denlats exp/tri3b_mmi || exit 1;
```
* Input: Folders exp/tri3b_ali, data/train, data/lang, exp/tri3b
* Output: Folders exp/tri3b_denlats and exp/tri3b_mmi
### Decode
```
steps/decode_fmllr.sh --config conf/decode.config --nj $nj --cmd "$decode_cmd" \
  --alignment-model exp/tri3b/final.alimdl --adapt-model exp/tri3b/final.mdl \
   exp/tri3b/graph data/test exp/tri3b_mmi/decode || exit 1;
```
* Input: Files exp/tri3b/final.alimdl and exp/tri3b/final.mdl; Folders exp/tri3b/graph and data/test
* Output: Folder exp/tri3b_mmi/decode
## Create DNN-HMM acoustic model
### nnet2
```
./local/run_nnet2.sh
```
