# BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding

## Jacob Devlin Ming-Wei Chang Kenton Lee Kristina Toutanova Google AI Language jacobdevlin,mingweichang,kentonl,kristout @google.com { }


## Abstract

We introduce a new language representation model called BERT, which stands for
**Bidirectional Encoder Representations from**
**Transformers. Unlike recent language repre-**
sentation models (Peters et al., 2018a; Radford et al., 2018), BERT is designed to pretrain deep bidirectional representations from
unlabeled text by jointly conditioning on both
left and right context in all layers. As a result, the pre-trained BERT model can be finetuned with just one additional output layer
to create state-of-the-art models for a wide
range of tasks, such as question answering and
language inference, without substantial taskspecific architecture modifications.

BERT is conceptually simple and empirically
powerful. It obtains new state-of-the-art results on eleven natural language processing
tasks, including pushing the GLUE score to
80.5% (7.7% point absolute improvement),
MultiNLI accuracy to 86.7% (4.6% absolute
improvement), SQuAD v1.1 question answering Test F1 to 93.2 (1.5 point absolute improvement) and SQuAD v2.0 Test F1 to 83.1
(5.1 point absolute improvement).

## 1 Introduction

Language model pre-training has been shown to
be effective for improving many natural language
processing tasks (Dai and Le, 2015; Peters et al.,
2018a; Radford et al., 2018; Howard and Ruder,
2018). These include sentence-level tasks such as
natural language inference (Bowman et al., 2015;
Williams et al., 2018) and paraphrasing (Dolan
and Brockett, 2005), which aim to predict the relationships between sentences by analyzing them
holistically, as well as token-level tasks such as
named entity recognition and question answering,
where models are required to produce fine-grained
output at the token level (Tjong Kim Sang and
De Meulder, 2003; Rajpurkar et al., 2016).


There are two existing strategies for applying pre-trained language representations to downstream tasks: feature-based and fine-tuning. The
feature-based approach, such as ELMo (Peters
et al., 2018a), uses task-specific architectures that
include the pre-trained representations as additional features. The fine-tuning approach, such as
the Generative Pre-trained Transformer (OpenAI
GPT) (Radford et al., 2018), introduces minimal
task-specific parameters, and is trained on the
downstream tasks by simply fine-tuning all pretrained parameters. The two approaches share the
same objective function during pre-training, where
they use unidirectional language models to learn
general language representations.
We argue that current techniques restrict the
power of the pre-trained representations, especially for the fine-tuning approaches. The major limitation is that standard language models are
unidirectional, and this limits the choice of architectures that can be used during pre-training. For
example, in OpenAI GPT, the authors use a left-toright architecture, where every token can only attend to previous tokens in the self-attention layers
of the Transformer (Vaswani et al., 2017). Such restrictions are sub-optimal for sentence-level tasks,
and could be very harmful when applying finetuning based approaches to token-level tasks such
as question answering, where it is crucial to incorporate context from both directions.
In this paper, we improve the fine-tuning based
approaches by proposing BERT: Bidirectional
**Encoder** **Representations** from **Transformers.**
BERT alleviates the previously mentioned unidirectionality constraint by using a “masked language model” (MLM) pre-training objective, inspired by the Cloze task (Taylor, 1953). The
masked language model randomly masks some of
the tokens from the input, and the objective is to
predict the original vocabulary id of the masked


4171


-----

word based only on its context. Unlike left-toright language model pre-training, the MLM objective enables the representation to fuse the left
and the right context, which allows us to pretrain a deep bidirectional Transformer. In addition to the masked language model, we also use
a “next sentence prediction” task that jointly pretrains text-pair representations. The contributions
of our paper are as follows:

- We demonstrate the importance of bidirectional
pre-training for language representations. Unlike Radford et al. (2018), which uses unidirectional language models for pre-training, BERT
uses masked language models to enable pretrained deep bidirectional representations. This
is also in contrast to Peters et al. (2018a), which
uses a shallow concatenation of independently
trained left-to-right and right-to-left LMs.

- We show that pre-trained representations reduce
the need for many heavily-engineered taskspecific architectures. BERT is the first finetuning based representation model that achieves
state-of-the-art performance on a large suite
of sentence-level and token-level tasks, outperforming many task-specific architectures.

- BERT advances the state of the art for eleven
NLP tasks. The code and pre-trained mod[els are available at https://github.com/](https://github.com/google-research/bert)
[google-research/bert.](https://github.com/google-research/bert)

## 2 Related Work

There is a long history of pre-training general language representations, and we briefly review the
most widely-used approaches in this section.

**2.1** **Unsupervised Feature-based Approaches**

Learning widely applicable representations of
words has been an active area of research for
decades, including non-neural (Brown et al., 1992;
Ando and Zhang, 2005; Blitzer et al., 2006) and
neural (Mikolov et al., 2013; Pennington et al.,
2014) methods. Pre-trained word embeddings
are an integral part of modern NLP systems, offering significant improvements over embeddings
learned from scratch (Turian et al., 2010). To pretrain word embedding vectors, left-to-right language modeling objectives have been used (Mnih
and Hinton, 2009), as well as objectives to discriminate correct from incorrect words in left and
right context (Mikolov et al., 2013).


These approaches have been generalized to
coarser granularities, such as sentence embeddings (Kiros et al., 2015; Logeswaran and Lee,
2018) or paragraph embeddings (Le and Mikolov,
2014). To train sentence representations, prior
work has used objectives to rank candidate next
sentences (Jernite et al., 2017; Logeswaran and
Lee, 2018), left-to-right generation of next sentence words given a representation of the previous
sentence (Kiros et al., 2015), or denoising autoencoder derived objectives (Hill et al., 2016).
ELMo and its predecessor (Peters et al., 2017,

2018a) generalize traditional word embedding research along a different dimension. They extract
_context-sensitive features from a left-to-right and a_
right-to-left language model. The contextual representation of each token is the concatenation of
the left-to-right and right-to-left representations.
When integrating contextual word embeddings
with existing task-specific architectures, ELMo
advances the state of the art for several major NLP
benchmarks (Peters et al., 2018a) including question answering (Rajpurkar et al., 2016), sentiment
analysis (Socher et al., 2013), and named entity
recognition (Tjong Kim Sang and De Meulder,
2003). Melamud et al. (2016) proposed learning
contextual representations through a task to predict a single word from both left and right context
using LSTMs. Similar to ELMo, their model is
feature-based and not deeply bidirectional. Fedus
et al. (2018) shows that the cloze task can be used
to improve the robustness of text generation models.

**2.2** **Unsupervised Fine-tuning Approaches**

As with the feature-based approaches, the first
works in this direction only pre-trained word embedding parameters from unlabeled text (Collobert and Weston, 2008).
More recently, sentence or document encoders
which produce contextual token representations
have been pre-trained from unlabeled text and
fine-tuned for a supervised downstream task (Dai
and Le, 2015; Howard and Ruder, 2018; Radford
et al., 2018). The advantage of these approaches
is that few parameters need to be learned from
scratch. At least partly due to this advantage,
OpenAI GPT (Radford et al., 2018) achieved previously state-of-the-art results on many sentencelevel tasks from the GLUE benchmark (Wang
et al., 2018a). Left-to-right language model

4172


-----

|T|’ M|
|---|---|

|E[CLS]|E 1|
|---|---|

|E N|E [SEP]|E’ 1|
|---|---|---|

|E[CLS]|E 1|
|---|---|

|E N|E [SEP]|E’ 1|
|---|---|---|

|E|’ M|
|---|---|

|To|kM|
|---|---|

|Col1|Col2|Col3|Col4|C T ... T T T’ 1 N [SEP] 1 BERT BERT E[CLS] E ... E E E 1’ 1 N [SEP]|... .|Col7|’ M ’ M|
|---|---|---|---|---|---|---|---|
|||||||.. To||


BERT BERT


T[SEP]


T1


T1’


C


TN


TM’


Figure 1: Overall pre-training and fine-tuning procedures for BERT. Apart from output layers, the same architectures are used in both pre-training and fine-tuning. The same pre-trained model parameters are used to initialize
models for different down-stream tasks. During fine-tuning, all parameters are fine-tuned. [CLS] is a special
symbol added in front of every input example, and [SEP] is a special separator token (e.g. separating questions/answers).


BERT


TN


T1


C


ing and auto-encoder objectives have been used
for pre-training such models (Howard and Ruder,
2018; Radford et al., 2018; Dai and Le, 2015).

**2.3** **Transfer Learning from Supervised Data**

There has also been work showing effective transfer from supervised tasks with large datasets, such
as natural language inference (Conneau et al.,
2017) and machine translation (McCann et al.,
2017). Computer vision research has also demonstrated the importance of transfer learning from
large pre-trained models, where an effective recipe
is to fine-tune models pre-trained with ImageNet (Deng et al., 2009; Yosinski et al., 2014).

## 3 BERT

We introduce BERT and its detailed implementation in this section. There are two steps in our
framework: pre-training and fine-tuning. During pre-training, the model is trained on unlabeled
data over different pre-training tasks. For finetuning, the BERT model is first initialized with
the pre-trained parameters, and all of the parameters are fine-tuned using labeled data from the
downstream tasks. Each downstream task has separate fine-tuned models, even though they are initialized with the same pre-trained parameters. The
question-answering example in Figure 1 will serve
as a running example for this section.
A distinctive feature of BERT is its unified architecture across different tasks. There is mini

T[SEP]


T1’


mal difference between the pre-trained architecture and the final downstream architecture.

**Model Architecture** BERT’s model architecture is a multi-layer bidirectional Transformer encoder based on the original implementation described in Vaswani et al. (2017) and released in
the tensor2tensor library.[1] Because the use
of Transformers has become common and our implementation is almost identical to the original,
we will omit an exhaustive background description of the model architecture and refer readers to
Vaswani et al. (2017) as well as excellent guides
such as “The Annotated Transformer.”[2]

In this work, we denote the number of layers
(i.e., Transformer blocks) as L, the hidden size as
_H, and the number of self-attention heads as A.[3]_

We primarily report results on two model sizes:
**BERTBASE (L=12, H=768, A=12, Total Param-**
eters=110M) and BERTLARGE (L=24, H=1024,
A=16, Total Parameters=340M).
BERTBASE was chosen to have the same model
size as OpenAI GPT for comparison purposes.
Critically, however, the BERT Transformer uses
bidirectional self-attention, while the GPT Transformer uses constrained self-attention where every
token can only attend to context to its left.[4]

1https://github.com/tensorflow/tensor2tensor
2http://nlp.seas.harvard.edu/2018/04/03/attention.html
3In all cases we set the feed-forward/filter size to be 4H,
i.e., 3072 for the H = 768 and 4096 for the H = 1024.
4We note that in the literature the bidirectional Trans

TM’


4173


-----

**Input/Output Representations** To make BERT
handle a variety of down-stream tasks, our input
representation is able to unambiguously represent
both a single sentence and a pair of sentences
(e.g., Question, Answer ) in one token sequence.
_⟨_ _⟩_
Throughout this work, a “sentence” can be an arbitrary span of contiguous text, rather than an actual
linguistic sentence. A “sequence” refers to the input token sequence to BERT, which may be a single sentence or two sentences packed together.
We use WordPiece embeddings (Wu et al.,

2016) with a 30,000 token vocabulary. The first
token of every sequence is always a special classification token ([CLS]). The final hidden state
corresponding to this token is used as the aggregate sequence representation for classification
tasks. Sentence pairs are packed together into a
single sequence. We differentiate the sentences in
two ways. First, we separate them with a special
token ([SEP]). Second, we add a learned embedding to every token indicating whether it belongs
to sentence A or sentence B. As shown in Figure 1,
we denote input embedding as E, the final hidden
vector of the special [CLS] token as C ∈ R[H],
and the final hidden vector for the i[th] input token
as Ti ∈ R[H] .
For a given token, its input representation is
constructed by summing the corresponding token,
segment, and position embeddings. A visualization of this construction can be seen in Figure 2.

**3.1** **Pre-training BERT**

Unlike Peters et al. (2018a) and Radford et al.
(2018), we do not use traditional left-to-right or
right-to-left language models to pre-train BERT.
Instead, we pre-train BERT using two unsupervised tasks, described in this section. This step
is presented in the left part of Figure 1.

**Task #1: Masked LM** Intuitively, it is reasonable to believe that a deep bidirectional model is
strictly more powerful than either a left-to-right
model or the shallow concatenation of a left-toright and a right-to-left model. Unfortunately,
standard conditional language models can only be
trained left-to-right or right-to-left, since bidirectional conditioning would allow each word to indirectly “see itself”, and the model could trivially
predict the target word in a multi-layered context.

former is often referred to as a “Transformer encoder” while
the left-context-only version is referred to as a “Transformer
decoder” since it can be used for text generation.


In order to train a deep bidirectional representation, we simply mask some percentage of the input
tokens at random, and then predict those masked
tokens. We refer to this procedure as a “masked
LM” (MLM), although it is often referred to as a
_Cloze task in the literature (Taylor, 1953). In this_
case, the final hidden vectors corresponding to the
mask tokens are fed into an output softmax over
the vocabulary, as in a standard LM. In all of our
experiments, we mask 15% of all WordPiece tokens in each sequence at random. In contrast to
denoising auto-encoders (Vincent et al., 2008), we
only predict the masked words rather than reconstructing the entire input.
Although this allows us to obtain a bidirectional pre-trained model, a downside is that we
are creating a mismatch between pre-training and
fine-tuning, since the [MASK] token does not appear during fine-tuning. To mitigate this, we do
not always replace “masked” words with the actual [MASK] token. The training data generator
chooses 15% of the token positions at random for
prediction. If the i-th token is chosen, we replace
the i-th token with (1) the [MASK] token 80% of
the time (2) a random token 10% of the time (3)
the unchanged i-th token 10% of the time. Then,
_Ti will be used to predict the original token with_
cross entropy loss. We compare variations of this
procedure in Appendix C.2.

**Task #2:** **Next Sentence Prediction (NSP)**
Many important downstream tasks such as Question Answering (QA) and Natural Language Inference (NLI) are based on understanding the rela_tionship between two sentences, which is not di-_
rectly captured by language modeling. In order
to train a model that understands sentence relationships, we pre-train for a binarized next sen_tence prediction task that can be trivially gener-_
ated from any monolingual corpus. Specifically,
when choosing the sentences A and B for each pretraining example, 50% of the time B is the actual
next sentence that follows A (labeled as IsNext),
and 50% of the time it is a random sentence from
the corpus (labeled as NotNext). As we show
in Figure 1, C is used for next sentence prediction (NSP).[5] Despite its simplicity, we demonstrate in Section 5.1 that pre-training towards this
task is very beneficial to both QA and NLI. [6]

5The final model achieves 97%-98% accuracy on NSP.
6The vector C is not a meaningful sentence representation
without fine-tuning, since it was trained with NSP.


4174


-----

Input [CLS] my dog is cute [SEP] he likes play ##ing [SEP]

|is|cute|[SEP]|
|---|---|---|

|he|likes|play|
|---|---|---|

|##ing|[SEP]|
|---|---|

|E is|E cute|E [SEP]|
|---|---|---|

|E he|E likes|E play|
|---|---|---|

|E ##ing|E [SEP]|
|---|---|


Segment
Embeddings

Position
Embeddings


EA EA EA EA EA EA EB EB EB EB EB

E0 E1 E2 E3 E4 E5 E6 E7 E8 E9 E10

|E A|E A|E A|
|---|---|---|

|E B|E B|E B|
|---|---|---|

|E B|E B|
|---|---|

|E 3|E 4|E 5|
|---|---|---|

|E 6|E 7|E 8|
|---|---|---|

|E 9|E 10|
|---|---|


Figure 2: BERT input representation. The input embeddings are the sum of the token embeddings, the segmentation embeddings and the position embeddings.


The NSP task is closely related to representationlearning objectives used in Jernite et al. (2017) and
Logeswaran and Lee (2018). However, in prior
work, only sentence embeddings are transferred to
down-stream tasks, where BERT transfers all parameters to initialize end-task model parameters.

**Pre-training data The pre-training procedure**
largely follows the existing literature on language
model pre-training. For the pre-training corpus we
use the BooksCorpus (800M words) (Zhu et al.,
2015) and English Wikipedia (2,500M words).
For Wikipedia we extract only the text passages
and ignore lists, tables, and headers. It is critical to use a document-level corpus rather than a
shuffled sentence-level corpus such as the Billion
Word Benchmark (Chelba et al., 2013) in order to
extract long contiguous sequences.

**3.2** **Fine-tuning BERT**

Fine-tuning is straightforward since the selfattention mechanism in the Transformer allows BERT to model many downstream tasks—
whether they involve single text or text pairs—by
swapping out the appropriate inputs and outputs.
For applications involving text pairs, a common
pattern is to independently encode text pairs before applying bidirectional cross attention, such
as Parikh et al. (2016); Seo et al. (2017). BERT
instead uses the self-attention mechanism to unify
these two stages, as encoding a concatenated text
pair with self-attention effectively includes bidi_rectional cross attention between two sentences._
For each task, we simply plug in the taskspecific inputs and outputs into BERT and finetune all the parameters end-to-end. At the input, sentence A and sentence B from pre-training
are analogous to (1) sentence pairs in paraphrasing, (2) hypothesis-premise pairs in entailment, (3)
question-passage pairs in question answering, and


(4) a degenerate text-∅ pair in text classification
or sequence tagging. At the output, the token representations are fed into an output layer for tokenlevel tasks, such as sequence tagging or question
answering, and the [CLS] representation is fed
into an output layer for classification, such as entailment or sentiment analysis.
Compared to pre-training, fine-tuning is relatively inexpensive. All of the results in the paper can be replicated in at most 1 hour on a single Cloud TPU, or a few hours on a GPU, starting
from the exact same pre-trained model.[7] We describe the task-specific details in the corresponding subsections of Section 4. More details can be
found in Appendix A.5.

## 4 Experiments

In this section, we present BERT fine-tuning results on 11 NLP tasks.

**4.1** **GLUE**

The General Language Understanding Evaluation
(GLUE) benchmark (Wang et al., 2018a) is a collection of diverse natural language understanding
tasks. Detailed descriptions of GLUE datasets are
included in Appendix B.1.
To fine-tune on GLUE, we represent the input
sequence (for single sentence or sentence pairs)
as described in Section 3, and use the final hidden vector C ∈ R[H] corresponding to the first
input token ([CLS]) as the aggregate representation. The only new parameters introduced during
fine-tuning are classification layer weights W
_∈_
R[K][×][H], where K is the number of labels. We compute a standard classification loss with C and W,
i.e., log(softmax(CW _[T]_ )).

7For example, the BERT SQuAD model can be trained in
around 30 minutes on a single Cloud TPU to achieve a Dev
F1 score of 91.0%.
[8See (10) in https://gluebenchmark.com/faq.](https://gluebenchmark.com/faq)


4175


-----

System MNLI-(m/mm) QQP QNLI SST-2 CoLA STS-B MRPC RTE **Average**
392k 363k 108k 67k 8.5k 5.7k 3.5k 2.5k             
Pre-OpenAI SOTA 80.6/80.1 66.1 82.3 93.2 35.0 81.0 86.0 61.7 74.0
BiLSTM+ELMo+Attn 76.4/76.1 64.8 79.8 90.4 36.0 73.3 84.9 56.8 71.0
OpenAI GPT 82.1/81.4 70.3 87.4 91.3 45.4 80.0 82.3 56.0 75.1

BERTBASE 84.6/83.4 71.2 90.5 93.5 52.1 85.8 88.9 66.4 79.6
BERTLARGE **86.7/85.9** **72.1** **92.7** **94.9** **60.5** **86.5** **89.3** **70.1** **82.1**

[Table 1: GLUE Test results, scored by the evaluation server (https://gluebenchmark.com/leaderboard).](https://gluebenchmark.com/leaderboard)
The number below each task denotes the number of training examples. The “Average” column is slightly different
than the official GLUE score, since we exclude the problematic WNLI set.[8] BERT and OpenAI GPT are singlemodel, single task. F1 scores are reported for QQP and MRPC, Spearman correlations are reported for STS-B, and
accuracy scores are reported for the other tasks. We exclude entries that use BERT as one of their components.


We use a batch size of 32 and fine-tune for 3
epochs over the data for all GLUE tasks. For each
task, we selected the best fine-tuning learning rate
(among 5e-5, 4e-5, 3e-5, and 2e-5) on the Dev set.
Additionally, for BERTLARGE we found that finetuning was sometimes unstable on small datasets,
so we ran several random restarts and selected the
best model on the Dev set. With random restarts,
we use the same pre-trained checkpoint but perform different fine-tuning data shuffling and classifier layer initialization.[9]

Results are presented in Table 1. Both
BERTBASE and BERTLARGE outperform all systems on all tasks by a substantial margin, obtaining
4.5% and 7.0% respective average accuracy improvement over the prior state of the art. Note that
BERTBASE and OpenAI GPT are nearly identical
in terms of model architecture apart from the attention masking. For the largest and most widely
reported GLUE task, MNLI, BERT obtains a 4.6%
absolute accuracy improvement. On the official
GLUE leaderboard[10], BERTLARGE obtains a score
of 80.5, compared to OpenAI GPT, which obtains
72.8 as of the date of writing.
We find that BERTLARGE significantly outperforms BERTBASE across all tasks, especially those
with very little training data. The effect of model
size is explored more thoroughly in Section 5.2.

**4.2** **SQuAD v1.1**

The Stanford Question Answering Dataset
(SQuAD v1.1) is a collection of 100k crowdsourced question/answer pairs (Rajpurkar et al.,
2016). Given a question and a passage from

9The GLUE data set distribution does not include the Test
labels, and we only made a single GLUE evaluation server
submission for each of BERTBASE and BERTLARGE.
10https://gluebenchmark.com/leaderboard


Wikipedia containing the answer, the task is to
predict the answer text span in the passage.
As shown in Figure 1, in the question answering task, we represent the input question and passage as a single packed sequence, with the question using the A embedding and the passage using
the B embedding. We only introduce a start vector S ∈ R[H] and an end vector E ∈ R[H] during
fine-tuning. The probability of word i being the
start of the answer span is computed as a dot product between Ti and S followed by a softmax over
_e[S][·][Ti]_
all of the words in the paragraph: Pi = �
_j_ _[e][S][·][Tj][ .]_

The analogous formula is used for the end of the
answer span. The score of a candidate span from
position i to position j is defined as S·Ti + E·Tj,
and the maximum scoring span where j _i is_
_≥_
used as a prediction. The training objective is the
sum of the log-likelihoods of the correct start and
end positions. We fine-tune for 3 epochs with a
learning rate of 5e-5 and a batch size of 32.
Table 2 shows top leaderboard entries as well
as results from top published systems (Seo et al.,
2017; Clark and Gardner, 2018; Peters et al.,
2018a; Hu et al., 2018). The top results from the
SQuAD leaderboard do not have up-to-date public
system descriptions available,[11] and are allowed to
use any public data when training their systems.
We therefore use modest data augmentation in
our system by first fine-tuning on TriviaQA (Joshi
et al., 2017) befor fine-tuning on SQuAD.
Our best performing system outperforms the top
leaderboard system by +1.5 F1 in ensembling and
+1.3 F1 as a single system. In fact, our single
BERT model outperforms the top ensemble system in terms of F1 score. Without TriviaQA fine
11QANet is described in Yu et al. (2018), but the system
has improved substantially after publication.


4176


-----

System Dev Test
EM F1 EM F1

Top Leaderboard Systems (Dec 10th, 2018)
Human  -  - 82.3 91.2
#1 Ensemble - nlnet  -  - 86.0 91.7
#2 Ensemble - QANet  -  - 84.5 90.5

Published
BiDAF+ELMo (Single)  - 85.6  - 85.8
R.M. Reader (Ensemble) 81.2 87.9 82.3 88.5

Ours
BERTBASE (Single) 80.8 88.5  -  BERTLARGE (Single) 84.1 90.9  -  BERTLARGE (Ensemble) 85.8 91.8  -  BERTLARGE (Sgl.+TriviaQA) **84.2 91.1 85.1 91.8**
BERTLARGE (Ens.+TriviaQA) 86.2 92.2 87.4 93.2

Table 2: SQuAD 1.1 results. The BERT ensemble
is 7x systems which use different pre-training checkpoints and fine-tuning seeds.

System Dev Test
EM F1 EM F1

Top Leaderboard Systems (Dec 10th, 2018)
Human 86.3 89.0 86.9 89.5
#1 Single - MIR-MRC (F-Net)  -  - 74.8 78.0
#2 Single - nlnet  -  - 74.2 77.1

Published
unet (Ensemble)  -  - 71.4 74.9
SLQA+ (Single)  - 71.4 74.4

Ours
BERTLARGE (Single) 78.7 81.9 80.0 83.1

Table 3: SQuAD 2.0 results. We exclude entries that
use BERT as one of their components.

tuning data, we only lose 0.1-0.4 F1, still outperforming all existing systems by a wide margin.[12]

**4.3** **SQuAD v2.0**

The SQuAD 2.0 task extends the SQuAD 1.1
problem definition by allowing for the possibility
that no short answer exists in the provided paragraph, making the problem more realistic.
We use a simple approach to extend the SQuAD
v1.1 BERT model for this task. We treat questions that do not have an answer as having an answer span with start and end at the [CLS] token. The probability space for the start and end
answer span positions is extended to include the
position of the [CLS] token. For prediction, we
compare the score of the no-answer span: snull =
_S_ _C + E_ _C to the score of the best non-null span_

_·_ _·_

12The TriviaQA data we used consists of paragraphs from
TriviaQA-Wiki formed of the first 400 tokens in documents,
that contain at least one of the provided possible answers.


System Dev Test

ESIM+GloVe 51.9 52.7
ESIM+ELMo 59.1 59.2
OpenAI GPT     - 78.0

BERTBASE 81.6     BERTLARGE **86.6 86.3**

Human (expert)[†]       - 85.0
Human (5 annotations)[†]       - 88.0

Table 4: SWAG Dev and Test accuracies. _[†]Human per-_
formance is measured with 100 samples, as reported in
the SWAG paper.

_sˆi,j = maxj≥iS·Ti + E·Tj. We predict a non-null_
answer when ˆsi,j > snull + τ, where the threshold τ is selected on the dev set to maximize F1.
We did not use TriviaQA data for this model. We
fine-tuned for 2 epochs with a learning rate of 5e-5
and a batch size of 48.
The results compared to prior leaderboard entries and top published work (Sun et al., 2018;
Wang et al., 2018b) are shown in Table 3, excluding systems that use BERT as one of their components. We observe a +5.1 F1 improvement over
the previous best system.

**4.4** **SWAG**

The Situations With Adversarial Generations
(SWAG) dataset contains 113k sentence-pair completion examples that evaluate grounded commonsense inference (Zellers et al., 2018). Given a sentence, the task is to choose the most plausible continuation among four choices.
When fine-tuning on the SWAG dataset, we
construct four input sequences, each containing
the concatenation of the given sentence (sentence
A) and a possible continuation (sentence B). The
only task-specific parameters introduced is a vector whose dot product with the [CLS] token representation C denotes a score for each choice
which is normalized with a softmax layer.
We fine-tune the model for 3 epochs with a
learning rate of 2e-5 and a batch size of 16. Results are presented in Table 4. BERTLARGE outperforms the authors’ baseline ESIM+ELMo system by +27.1% and OpenAI GPT by 8.3%.

## 5 Ablation Studies

In this section, we perform ablation experiments
over a number of facets of BERT in order to better
understand their relative importance. Additional


4177


-----

Dev Set
Tasks MNLI-m QNLI MRPC SST-2 SQuAD
(Acc) (Acc) (Acc) (Acc) (F1)

BERTBASE 84.4 88.4 86.7 92.7 88.5
No NSP 83.9 84.9 86.5 92.6 87.9
LTR & No NSP 82.1 84.3 77.5 92.1 77.8
+ BiLSTM 82.1 84.1 75.7 91.6 84.9

Table 5: Ablation over the pre-training tasks using the
BERTBASE architecture. “No NSP” is trained without
the next sentence prediction task. “LTR & No NSP” is
trained as a left-to-right LM without the next sentence
prediction, like OpenAI GPT. “+ BiLSTM” adds a randomly initialized BiLSTM on top of the “LTR + No
NSP” model during fine-tuning.

ablation studies can be found in Appendix C.

**5.1** **Effect of Pre-training Tasks**

We demonstrate the importance of the deep bidirectionality of BERT by evaluating two pretraining objectives using exactly the same pretraining data, fine-tuning scheme, and hyperparameters as BERTBASE:

**No NSP: A bidirectional model which is trained**
using the “masked LM” (MLM) but without the
“next sentence prediction” (NSP) task.
**LTR & No NSP: A left-context-only model which**
is trained using a standard Left-to-Right (LTR)
LM, rather than an MLM. The left-only constraint
was also applied at fine-tuning, because removing
it introduced a pre-train/fine-tune mismatch that
degraded downstream performance. Additionally,
this model was pre-trained without the NSP task.
This is directly comparable to OpenAI GPT, but
using our larger training dataset, our input representation, and our fine-tuning scheme.
We first examine the impact brought by the NSP
task. In Table 5, we show that removing NSP
hurts performance significantly on QNLI, MNLI,
and SQuAD 1.1. Next, we evaluate the impact
of training bidirectional representations by comparing “No NSP” to “LTR & No NSP”. The LTR
model performs worse than the MLM model on all
tasks, with large drops on MRPC and SQuAD.
For SQuAD it is intuitively clear that a LTR
model will perform poorly at token predictions,
since the token-level hidden states have no rightside context. In order to make a good faith attempt at strengthening the LTR system, we added
a randomly initialized BiLSTM on top. This does
significantly improve results on SQuAD, but the


results are still far worse than those of the pretrained bidirectional models. The BiLSTM hurts
performance on the GLUE tasks.
We recognize that it would also be possible to
train separate LTR and RTL models and represent
each token as the concatenation of the two models, as ELMo does. However: (a) this is twice as
expensive as a single bidirectional model; (b) this
is non-intuitive for tasks like QA, since the RTL
model would not be able to condition the answer
on the question; (c) this it is strictly less powerful
than a deep bidirectional model, since it can use
both left and right context at every layer.

**5.2** **Effect of Model Size**

In this section, we explore the effect of model size
on fine-tuning task accuracy. We trained a number
of BERT models with a differing number of layers,
hidden units, and attention heads, while otherwise
using the same hyperparameters and training procedure as described previously.
Results on selected GLUE tasks are shown in
Table 6. In this table, we report the average Dev
Set accuracy from 5 random restarts of fine-tuning.
We can see that larger models lead to a strict accuracy improvement across all four datasets, even
for MRPC which only has 3,600 labeled training examples, and is substantially different from
the pre-training tasks. It is also perhaps surprising that we are able to achieve such significant
improvements on top of models which are already quite large relative to the existing literature.
For example, the largest Transformer explored in
Vaswani et al. (2017) is (L=6, H=1024, A=16)
with 100M parameters for the encoder, and the
largest Transformer we have found in the literature
is (L=64, H=512, A=2) with 235M parameters
(Al-Rfou et al., 2018). By contrast, BERTBASE
contains 110M parameters and BERTLARGE contains 340M parameters.
It has long been known that increasing the
model size will lead to continual improvements
on large-scale tasks such as machine translation
and language modeling, which is demonstrated
by the LM perplexity of held-out training data
shown in Table 6. However, we believe that
this is the first work to demonstrate convincingly that scaling to extreme model sizes also
leads to large improvements on very small scale
tasks, provided that the model has been sufficiently pre-trained. Peters et al. (2018b) presented


4178


-----

mixed results on the downstream task impact of
increasing the pre-trained bi-LM size from two
to four layers and Melamud et al. (2016) mentioned in passing that increasing hidden dimension size from 200 to 600 helped, but increasing
further to 1,000 did not bring further improvements. Both of these prior works used a featurebased approach — we hypothesize that when the
model is fine-tuned directly on the downstream
tasks and uses only a very small number of randomly initialized additional parameters, the taskspecific models can benefit from the larger, more
expressive pre-trained representations even when
downstream task data is very small.

**5.3** **Feature-based Approach with BERT**

All of the BERT results presented so far have used
the fine-tuning approach, where a simple classification layer is added to the pre-trained model, and
all parameters are jointly fine-tuned on a downstream task. However, the feature-based approach,
where fixed features are extracted from the pretrained model, has certain advantages. First, not
all tasks can be easily represented by a Transformer encoder architecture, and therefore require
a task-specific model architecture to be added.
Second, there are major computational benefits
to pre-compute an expensive representation of the
training data once and then run many experiments
with cheaper models on top of this representation.
In this section, we compare the two approaches
by applying BERT to the CoNLL-2003 Named
Entity Recognition (NER) task (Tjong Kim Sang
and De Meulder, 2003). In the input to BERT, we
use a case-preserving WordPiece model, and we
include the maximal document context provided
by the data. Following standard practice, we formulate this as a tagging task but do not use a CRF

Hyperparams Dev Set Accuracy

#L #H #A LM (ppl) MNLI-m MRPC SST-2

3 768 12 5.84 77.9 79.8 88.4
6 768 3 5.24 80.6 82.2 90.7
6 768 12 4.68 81.9 84.8 91.3
12 768 12 3.99 84.4 86.7 92.9
12 1024 16 3.54 85.7 86.9 93.3
24 1024 16 3.23 86.6 87.8 93.7

Table 6: Ablation over BERT model size. #L = the
number of layers; #H = hidden size; #A = number of attention heads. “LM (ppl)” is the masked LM perplexity
of held-out training data.


System Dev F1 Test F1

ELMo (Peters et al., 2018a) 95.7 92.2
CVT (Clark et al., 2018)  - 92.6
CSE (Akbik et al., 2018)  - **93.1**

Fine-tuning approach
BERTLARGE 96.6 92.8
BERTBASE 96.4 92.4

Feature-based approach (BERTBASE)
Embeddings 91.0   Second-to-Last Hidden 95.6   Last Hidden 94.9   Weighted Sum Last Four Hidden 95.9   Concat Last Four Hidden 96.1   Weighted Sum All 12 Layers 95.5   
Table 7: CoNLL-2003 Named Entity Recognition results. Hyperparameters were selected using the Dev
set. The reported Dev and Test scores are averaged over
5 random restarts using those hyperparameters.

layer in the output. We use the representation of
the first sub-token as the input to the token-level
classifier over the NER label set.

To ablate the fine-tuning approach, we apply the
feature-based approach by extracting the activations from one or more layers without fine-tuning
any parameters of BERT. These contextual embeddings are used as input to a randomly initialized two-layer 768-dimensional BiLSTM before
the classification layer.

Results are presented in Table 7. BERTLARGE
performs competitively with state-of-the-art methods. The best performing method concatenates the
token representations from the top four hidden layers of the pre-trained Transformer, which is only
0.3 F1 behind fine-tuning the entire model. This
demonstrates that BERT is effective for both finetuning and feature-based approaches.

## 6 Conclusion

Recent empirical improvements due to transfer
learning with language models have demonstrated
that rich, unsupervised pre-training is an integral
part of many language understanding systems. In
particular, these results enable even low-resource
tasks to benefit from deep unidirectional architectures. Our major contribution is further generalizing these findings to deep bidirectional architectures, allowing the same pre-trained model to successfully tackle a broad set of NLP tasks.


4179


-----

## References

Alan Akbik, Duncan Blythe, and Roland Vollgraf.
2018. Contextual string embeddings for sequence
labeling. In Proceedings of the 27th International
_Conference on Computational Linguistics, pages_
1638–1649.

Rami Al-Rfou, Dokook Choe, Noah Constant, Mandy
Guo, and Llion Jones. 2018. Character-level language modeling with deeper self-attention. _arXiv_
_preprint arXiv:1808.04444._

Rie Kubota Ando and Tong Zhang. 2005. A framework
for learning predictive structures from multiple tasks
and unlabeled data. Journal of Machine Learning
_Research, 6(Nov):1817–1853._

Luisa Bentivogli, Bernardo Magnini, Ido Dagan,
Hoa Trang Dang, and Danilo Giampiccolo. 2009.
The fifth PASCAL recognizing textual entailment
challenge. In TAC. NIST.

John Blitzer, Ryan McDonald, and Fernando Pereira.
2006. Domain adaptation with structural correspondence learning. In Proceedings of the 2006 confer_ence on empirical methods in natural language pro-_
_cessing, pages 120–128. Association for Computa-_
tional Linguistics.

Samuel R. Bowman, Gabor Angeli, Christopher Potts,
and Christopher D. Manning. 2015. A large annotated corpus for learning natural language inference.
In EMNLP. Association for Computational Linguistics.

Peter F Brown, Peter V Desouza, Robert L Mercer,
Vincent J Della Pietra, and Jenifer C Lai. 1992.
Class-based n-gram models of natural language.
_Computational linguistics, 18(4):467–479._

Daniel Cer, Mona Diab, Eneko Agirre, Inigo LopezGazpio, and Lucia Specia. 2017. [Semeval-2017](https://doi.org/10.18653/v1/S17-2001)
[task 1: Semantic textual similarity multilingual and](https://doi.org/10.18653/v1/S17-2001)
[crosslingual focused evaluation.](https://doi.org/10.18653/v1/S17-2001) In Proceedings
_of the 11th International Workshop on Semantic_
_Evaluation (SemEval-2017), pages 1–14, Vancou-_
ver, Canada. Association for Computational Linguistics.

Ciprian Chelba, Tomas Mikolov, Mike Schuster, Qi Ge,
Thorsten Brants, Phillipp Koehn, and Tony Robinson. 2013. One billion word benchmark for measuring progress in statistical language modeling. arXiv
_preprint arXiv:1312.3005._

Z. Chen, H. Zhang, X. Zhang, and L. Zhao. 2018.

[Quora question pairs.](https://data.quora.com/First-Quora-Dataset-Release-Question-Pairs)

Christopher Clark and Matt Gardner. 2018. Simple
and effective multi-paragraph reading comprehension. In ACL.


Kevin Clark, Minh-Thang Luong, Christopher D Manning, and Quoc Le. 2018. Semi-supervised sequence modeling with cross-view training. In Pro_ceedings of the 2018 Conference on Empirical Meth-_
_ods in Natural Language Processing, pages 1914–_
1925.

Ronan Collobert and Jason Weston. 2008. A unified
architecture for natural language processing: Deep
neural networks with multitask learning. In Pro_ceedings of the 25th international conference on_
_Machine learning, pages 160–167. ACM._

Alexis Conneau, Douwe Kiela, Holger Schwenk, Lo¨ıc
Barrault, and Antoine Bordes. 2017. [Supervised](https://www.aclweb.org/anthology/D17-1070)
[learning of universal sentence representations from](https://www.aclweb.org/anthology/D17-1070)
[natural language inference data. In Proceedings of](https://www.aclweb.org/anthology/D17-1070)
_the 2017 Conference on Empirical Methods in Nat-_
_ural Language Processing, pages 670–680, Copen-_
hagen, Denmark. Association for Computational
Linguistics.

Andrew M Dai and Quoc V Le. 2015. Semi-supervised
sequence learning. In Advances in neural informa_tion processing systems, pages 3079–3087._

J. Deng, W. Dong, R. Socher, L.-J. Li, K. Li, and L. FeiFei. 2009. ImageNet: A Large-Scale Hierarchical
Image Database. In CVPR09.

William B Dolan and Chris Brockett. 2005. Automatically constructing a corpus of sentential paraphrases.
In Proceedings of the Third International Workshop
_on Paraphrasing (IWP2005)._

William Fedus, Ian Goodfellow, and Andrew M Dai.
2018. Maskgan: Better text generation via filling in
the . arXiv preprint arXiv:1801.07736.

Dan Hendrycks and Kevin Gimpel. 2016. [Bridging](http://arxiv.org/abs/1606.08415)
[nonlinearities and stochastic regularizers with gaus-](http://arxiv.org/abs/1606.08415)
[sian error linear units. CoRR, abs/1606.08415.](http://arxiv.org/abs/1606.08415)

Felix Hill, Kyunghyun Cho, and Anna Korhonen. 2016.
Learning distributed representations of sentences
from unlabelled data. In Proceedings of the 2016
_Conference of the North American Chapter of the_
_Association for Computational Linguistics: Human_
_Language Technologies. Association for Computa-_
tional Linguistics.

[Jeremy Howard and Sebastian Ruder. 2018. Universal](http://arxiv.org/abs/1801.06146)
[language model fine-tuning for text classification. In](http://arxiv.org/abs/1801.06146)
_ACL. Association for Computational Linguistics._

Minghao Hu, Yuxing Peng, Zhen Huang, Xipeng Qiu,
Furu Wei, and Ming Zhou. 2018. Reinforced
mnemonic reader for machine reading comprehension. In IJCAI.

Yacine Jernite, Samuel R. Bowman, and David Son[tag. 2017. Discourse-based objectives for fast un-](http://arxiv.org/abs/1705.00557)
[supervised sentence representation learning. CoRR,](http://arxiv.org/abs/1705.00557)
abs/1705.00557.


4180


-----

Mandar Joshi, Eunsol Choi, Daniel S Weld, and Luke
Zettlemoyer. 2017. Triviaqa: A large scale distantly
supervised challenge dataset for reading comprehension. In ACL.

Ryan Kiros, Yukun Zhu, Ruslan R Salakhutdinov,
Richard Zemel, Raquel Urtasun, Antonio Torralba,
and Sanja Fidler. 2015. Skip-thought vectors. In
_Advances in neural information processing systems,_
pages 3294–3302.

Quoc Le and Tomas Mikolov. 2014. Distributed representations of sentences and documents. In Inter_national Conference on Machine Learning, pages_
1188–1196.

Hector J Levesque, Ernest Davis, and Leora Morgenstern. 2011. The winograd schema challenge. In
_Aaai spring symposium: Logical formalizations of_
_commonsense reasoning, volume 46, page 47._

[Lajanugen Logeswaran and Honglak Lee. 2018. An](https://openreview.net/forum?id=rJvJXZb0W)
[efficient framework for learning sentence represen-](https://openreview.net/forum?id=rJvJXZb0W)
[tations.](https://openreview.net/forum?id=rJvJXZb0W) In International Conference on Learning
_Representations._

Bryan McCann, James Bradbury, Caiming Xiong, and
Richard Socher. 2017. Learned in translation: Contextualized word vectors. In NIPS.

Oren Melamud, Jacob Goldberger, and Ido Dagan.
2016. context2vec: Learning generic context embedding with bidirectional LSTM. In CoNLL.

Tomas Mikolov, Ilya Sutskever, Kai Chen, Greg S Corrado, and Jeff Dean. 2013. Distributed representations of words and phrases and their compositionality. In Advances in Neural Information Processing
_Systems 26, pages 3111–3119. Curran Associates,_
Inc.

[Andriy Mnih and Geoffrey E Hinton. 2009. A scal-](http://papers.nips.cc/paper/3583-a-scalable-hierarchical-distributed-language-model.pdf)
[able hierarchical distributed language model.](http://papers.nips.cc/paper/3583-a-scalable-hierarchical-distributed-language-model.pdf) In
D. Koller, D. Schuurmans, Y. Bengio, and L. Bottou, editors, Advances in Neural Information Pro_cessing Systems 21, pages 1081–1088. Curran As-_
sociates, Inc.

Ankur P Parikh, Oscar T¨ackstr¨om, Dipanjan Das, and
Jakob Uszkoreit. 2016. A decomposable attention
model for natural language inference. In EMNLP.

Jeffrey Pennington, Richard Socher, and Christo[pher D. Manning. 2014. Glove: Global vectors for](http://www.aclweb.org/anthology/D14-1162)
[word representation. In Empirical Methods in Nat-](http://www.aclweb.org/anthology/D14-1162)
_ural Language Processing (EMNLP), pages 1532–_
1543.

Matthew Peters, Waleed Ammar, Chandra Bhagavatula, and Russell Power. 2017. Semi-supervised sequence tagging with bidirectional language models.
In ACL.

Matthew Peters, Mark Neumann, Mohit Iyyer, Matt
Gardner, Christopher Clark, Kenton Lee, and Luke
Zettlemoyer. 2018a. Deep contextualized word representations. In NAACL.


Matthew Peters, Mark Neumann, Luke Zettlemoyer,
and Wen-tau Yih. 2018b. Dissecting contextual
word embeddings: Architecture and representation.
In Proceedings of the 2018 Conference on Empiri_cal Methods in Natural Language Processing, pages_
1499–1509.

Alec Radford, Karthik Narasimhan, Tim Salimans, and
Ilya Sutskever. 2018. Improving language understanding with unsupervised learning. Technical report, OpenAI.

Pranav Rajpurkar, Jian Zhang, Konstantin Lopyrev, and
Percy Liang. 2016. Squad: 100,000+ questions for
machine comprehension of text. In Proceedings of
_the 2016 Conference on Empirical Methods in Nat-_
_ural Language Processing, pages 2383–2392._

Minjoon Seo, Aniruddha Kembhavi, Ali Farhadi, and
Hannaneh Hajishirzi. 2017. Bidirectional attention
flow for machine comprehension. In ICLR.

Richard Socher, Alex Perelygin, Jean Wu, Jason
Chuang, Christopher D Manning, Andrew Ng, and
Christopher Potts. 2013. Recursive deep models
for semantic compositionality over a sentiment treebank. In Proceedings of the 2013 conference on
_empirical methods in natural language processing,_
pages 1631–1642.

Fu Sun, Linyang Li, Xipeng Qiu, and Yang Liu.
2018. U-net: Machine reading comprehension
with unanswerable questions. _arXiv preprint_
_arXiv:1810.06638._

Wilson L Taylor. 1953. Cloze procedure: A new
tool for measuring readability. Journalism Bulletin,
30(4):415–433.

Erik F Tjong Kim Sang and Fien De Meulder.
2003. Introduction to the conll-2003 shared task:
Language-independent named entity recognition. In
_CoNLL._

Joseph Turian, Lev Ratinov, and Yoshua Bengio. 2010.
Word representations: A simple and general method
for semi-supervised learning. In Proceedings of the
_48th Annual Meeting of the Association for Compu-_
_tational Linguistics, ACL ’10, pages 384–394._

Ashish Vaswani, Noam Shazeer, Niki Parmar, Jakob
Uszkoreit, Llion Jones, Aidan N Gomez, Lukasz
Kaiser, and Illia Polosukhin. 2017. Attention is all
you need. In Advances in Neural Information Pro_cessing Systems, pages 6000–6010._

Pascal Vincent, Hugo Larochelle, Yoshua Bengio, and
Pierre-Antoine Manzagol. 2008. Extracting and
composing robust features with denoising autoencoders. In Proceedings of the 25th international
_conference on Machine learning, pages 1096–1103._
ACM.

Alex Wang, Amanpreet Singh, Julian Michael, Felix Hill, Omer Levy, and Samuel Bowman. 2018a.
Glue: A multi-task benchmark and analysis platform


4181


-----

for natural language understanding. In Proceedings
_of the 2018 EMNLP Workshop BlackboxNLP: An-_
_alyzing and Interpreting Neural Networks for NLP,_
pages 353–355.

Wei Wang, Ming Yan, and Chen Wu. 2018b. Multigranularity hierarchical attention fusion networks
for reading comprehension and question answering.
In Proceedings of the 56th Annual Meeting of the As_sociation for Computational Linguistics (Volume 1:_
_Long Papers). Association for Computational Lin-_
guistics.

Alex Warstadt, Amanpreet Singh, and Samuel R Bowman. 2018. Neural network acceptability judgments. arXiv preprint arXiv:1805.12471.

Adina Williams, Nikita Nangia, and Samuel R Bowman. 2018. A broad-coverage challenge corpus
for sentence understanding through inference. In
_NAACL._

Yonghui Wu, Mike Schuster, Zhifeng Chen, Quoc V
Le, Mohammad Norouzi, Wolfgang Macherey,
Maxim Krikun, Yuan Cao, Qin Gao, Klaus
Macherey, et al. 2016. Google’s neural machine translation system: Bridging the gap between
human and machine translation. _arXiv preprint_
_arXiv:1609.08144._

Jason Yosinski, Jeff Clune, Yoshua Bengio, and Hod
Lipson. 2014. How transferable are features in deep
neural networks? In Advances in neural information
_processing systems, pages 3320–3328._

Adams Wei Yu, David Dohan, Minh-Thang Luong, Rui
Zhao, Kai Chen, Mohammad Norouzi, and Quoc V
Le. 2018. QANet: Combining local convolution
with global self-attention for reading comprehension. In ICLR.

Rowan Zellers, Yonatan Bisk, Roy Schwartz, and Yejin
Choi. 2018. Swag: A large-scale adversarial dataset
for grounded commonsense inference. In Proceed_ings of the 2018 Conference on Empirical Methods_
_in Natural Language Processing (EMNLP)._

Yukun Zhu, Ryan Kiros, Rich Zemel, Ruslan Salakhutdinov, Raquel Urtasun, Antonio Torralba, and Sanja
Fidler. 2015. Aligning books and movies: Towards
story-like visual explanations by watching movies
and reading books. In Proceedings of the IEEE
_international conference on computer vision, pages_
19–27.

## Appendix for “BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding”

We organize the appendix into three sections:

  - Additional implementation details for BERT
are presented in Appendix A;



  - Additional details for our experiments are
presented in Appendix B; and

  - Additional ablation studies are presented in
Appendix C.

We present additional ablation studies for
BERT including:

**– Effect of Number of Training Steps; and**
**– Ablation for Different Masking Proce-**
dures.

## A Additional Details for BERT

**A.1** **Illustration of the Pre-training Tasks**

We provide examples of the pre-training tasks in
the following.

**Masked LM and the Masking Procedure** Assuming the unlabeled sentence is my dog is

hairy, and during the random masking procedure
we chose the 4-th token (which corresponding to

hairy), our masking procedure can be further illustrated by

  - 80% of the time: Replace the word with the

[MASK] token, e.g., my dog is hairy →

my dog is [MASK]

  - 10% of the time: Replace the word with a
random word, e.g., my dog is hairy → my

dog is apple

 - 10% of the time: Keep the word unchanged, e.g., my dog is hairy → my dog

is hairy. The purpose of this is to bias the
representation towards the actual observed
word.

The advantage of this procedure is that the
Transformer encoder does not know which words
it will be asked to predict or which have been replaced by random words, so it is forced to keep
a distributional contextual representation of ev_ery input token._ Additionally, because random
replacement only occurs for 1.5% of all tokens
(i.e., 10% of 15%), this does not seem to harm
the model’s language understanding capability. In
Section C.2, we evaluate the impact this procedure.
Compared to standard langauge model training,
the masked LM only make predictions on 15% of
tokens in each batch, which suggests that more
pre-training steps may be required for the model


4182


-----

Figure 3: Differences in pre-training model architectures. BERT uses a bidirectional Transformer. OpenAI GPT
uses a left-to-right Transformer. ELMo uses the concatenation of independently trained left-to-right and right-toleft LSTMs to generate features for downstream tasks. Among the three, only BERT representations are jointly
conditioned on both left and right context in all layers. In addition to the architecture differences, BERT and
OpenAI GPT are fine-tuning approaches, while ELMo is a feature-based approach.


to converge. In Section C.1 we demonstrate that
MLM does converge marginally slower than a leftto-right model (which predicts every token), but
the empirical improvements of the MLM model
far outweigh the increased training cost.

**Next Sentence Prediction** The next sentence
prediction task can be illustrated in the following
examples.

Input = [CLS] the man went to [MASK] store [SEP]

he bought a gallon [MASK] milk [SEP]

Label = IsNext

Input = [CLS] the man [MASK] to the store [SEP]

penguin [MASK] are flight ##less birds [SEP]

Label = NotNext

**A.2** **Pre-training Procedure**

To generate each training input sequence, we sample two spans of text from the corpus, which we
refer to as “sentences” even though they are typically much longer than single sentences (but can
be shorter also). The first sentence receives the A
embedding and the second receives the B embedding. 50% of the time B is the actual next sentence
that follows A and 50% of the time it is a random
sentence, which is done for the “next sentence prediction” task. They are sampled such that the combined length is 512 tokens. The LM masking is
_≤_
applied after WordPiece tokenization with a uniform masking rate of 15%, and no special consideration given to partial word pieces.
We train with batch size of 256 sequences (256
sequences * 512 tokens = 128,000 tokens/batch)
for 1,000,000 steps, which is approximately 40


epochs over the 3.3 billion word corpus. We
use Adam with learning rate of 1e-4, β1 = 0.9,
_β2 = 0.999, L2 weight decay of 0.01, learning_
rate warmup over the first 10,000 steps, and linear
decay of the learning rate. We use a dropout probability of 0.1 on all layers. We use a gelu activation (Hendrycks and Gimpel, 2016) rather than
the standard relu, following OpenAI GPT. The
training loss is the sum of the mean masked LM
likelihood and the mean next sentence prediction
likelihood.
Training of BERTBASE was performed on 4
Cloud TPUs in Pod configuration (16 TPU chips
total).[13] Training of BERTLARGE was performed
on 16 Cloud TPUs (64 TPU chips total). Each pretraining took 4 days to complete.
Longer sequences are disproportionately expensive because attention is quadratic to the sequence
length. To speed up pretraing in our experiments,
we pre-train the model with sequence length of
128 for 90% of the steps. Then, we train the rest
10% of the steps of sequence of 512 to learn the
positional embeddings.

**A.3** **Fine-tuning Procedure**

For fine-tuning, most model hyperparameters are
the same as in pre-training, with the exception of
the batch size, learning rate, and number of training epochs. The dropout probability was always
kept at 0.1. The optimal hyperparameter values
are task-specific, but we found the following range
of possible values to work well across all tasks:

  - Batch size: 16, 32

13https://cloudplatform.googleblog.com/2018/06/CloudTPU-now-offers-preemptible-pricing-and-globalavailability.html


4183


-----

  - Learning rate (Adam): 5e-5, 3e-5, 2e-5

  - Number of epochs: 2, 3, 4

We also observed that large data sets (e.g.,
100k+ labeled training examples) were far less
sensitive to hyperparameter choice than small data
sets. Fine-tuning is typically very fast, so it is reasonable to simply run an exhaustive search over
the above parameters and choose the model that
performs best on the development set.

**A.4** **Comparison of BERT, ELMo,and**
**OpenAI GPT**

Here we studies the differences in recent popular
representation learning models including ELMo,
OpenAI GPT and BERT. The comparisons between the model architectures are shown visually
in Figure 3. Note that in addition to the architecture differences, BERT and OpenAI GPT are finetuning approaches, while ELMo is a feature-based
approach.
The most comparable existing pre-training
method to BERT is OpenAI GPT, which trains a
left-to-right Transformer LM on a large text corpus. In fact, many of the design decisions in BERT
were intentionally made to make it as close to
GPT as possible so that the two methods could be
minimally compared. The core argument of this
work is that the bi-directionality and the two pretraining tasks presented in Section 3.1 account for
the majority of the empirical improvements, but
we do note that there are several other differences
between how BERT and GPT were trained:

  - GPT is trained on the BooksCorpus (800M
words); BERT is trained on the BooksCorpus (800M words) and Wikipedia (2,500M
words).

  - GPT uses a sentence separator ([SEP]) and
classifier token ([CLS]) which are only introduced at fine-tuning time; BERT learns

[SEP], [CLS] and sentence A/B embeddings during pre-training.

  - GPT was trained for 1M steps with a batch
size of 32,000 words; BERT was trained for
1M steps with a batch size of 128,000 words.

  - GPT used the same learning rate of 5e-5 for
all fine-tuning experiments; BERT chooses a
task-specific fine-tuning learning rate which
performs the best on the development set.


To isolate the effect of these differences, we perform ablation experiments in Section 5.1 which
demonstrate that the majority of the improvements
are in fact coming from the two pre-training tasks
and the bidirectionality they enable.

**A.5** **Illustrations of Fine-tuning on Different**
**Tasks**

The illustration of fine-tuning BERT on different
tasks can be seen in Figure 4. Our task-specific
models are formed by incorporating BERT with
one additional output layer, so a minimal number of parameters need to be learned from scratch.
Among the tasks, (a) and (b) are sequence-level
tasks while (c) and (d) are token-level tasks. In
the figure, E represents the input embedding, Ti
represents the contextual representation of token i,

[CLS] is the special symbol for classification output, and [SEP] is the special symbol to separate
non-consecutive token sequences.

## B Detailed Experimental Setup

**B.1** **Detailed Descriptions for the GLUE**
**Benchmark Experiments.**

The GLUE benchmark includes the following
datasets, the descriptions of which were originally
summarized in Wang et al. (2018a):

**MNLI** Multi-Genre Natural Language Inference
is a large-scale, crowdsourced entailment classification task (Williams et al., 2018). Given a pair of
sentences, the goal is to predict whether the second sentence is an entailment, contradiction, or
_neutral with respect to the first one._

**QQP** Quora Question Pairs is a binary classification task where the goal is to determine if two
questions asked on Quora are semantically equivalent (Chen et al., 2018).

**QNLI** Question Natural Language Inference is
a version of the Stanford Question Answering
Dataset (Rajpurkar et al., 2016) which has been
converted to a binary classification task (Wang
et al., 2018a). The positive examples are (question, sentence) pairs which do contain the correct
answer, and the negative examples are (question,
sentence) from the same paragraph which do not
contain the answer.

**SST-2** The Stanford Sentiment Treebank is a
binary single-sentence classification task consisting of sentences extracted from movie reviews


4184


-----

|E[CLS]|E 1|
|---|---|

|E N|E [SEP]|E’ 1|
|---|---|---|

|E [CLS]|E 1|E 2|
|---|---|---|

|C T 1 T 2 ... T N BERT E [CLS] E 1 E 2 ... E N|Col2|Col3|Col4|
|---|---|---|---|
|[[CCLLSS]]|TTookk 11|Tok 2|Tok N|

|E[CLS]|E 1|
|---|---|

|E N|E [SEP]|E’ 1|
|---|---|---|

|E [CLS]|E 1|E 2|
|---|---|---|


BERT


BERT


T1


TN


T2


C


T[SEP]


T1


T1’


T1


T1’


C


Figure 4: Illustrations of Fine-tuning BERT on Different Tasks.


BERT


T2


TN


T[SEP]


TM’


TM’


with human annotations of their sentiment (Socher
et al., 2013).

**CoLA** The Corpus of Linguistic Acceptability is
a binary single-sentence classification task, where
the goal is to predict whether an English sentence
is linguistically “acceptable” or not (Warstadt
et al., 2018).

**STS-B** The Semantic Textual Similarity Benchmark is a collection of sentence pairs drawn from
news headlines and other sources (Cer et al.,
2017). They were annotated with a score from 1
to 5 denoting how similar the two sentences are in
terms of semantic meaning.

**MRPC** Microsoft Research Paraphrase Corpus
consists of sentence pairs automatically extracted
from online news sources, with human annotations
for whether the sentences in the pair are semantically equivalent (Dolan and Brockett, 2005).


BERT


C


TN


**RTE** Recognizing Textual Entailment is a binary entailment task similar to MNLI, but with
much less training data (Bentivogli et al., 2009).[14]

**WNLI** Winograd NLI is a small natural language inference dataset (Levesque et al., 2011).
The GLUE webpage notes that there are issues
with the construction of this dataset, [15] and every
trained system that’s been submitted to GLUE has
performed worse than the 65.1 baseline accuracy
of predicting the majority class. We therefore exclude this set to be fair to OpenAI GPT. For our
GLUE submission, we always predicted the majority class.

14Note that we only report single-task fine-tuning results
in this paper. A multitask fine-tuning approach could potentially push the performance even further. For example, we
did observe substantial improvements on RTE from multitask training with MNLI.
[15https://gluebenchmark.com/faq](https://gluebenchmark.com/faq)


T1


C


4185


TN


-----

## C Additional Ablation Studies

**C.1** **Effect of Number of Training Steps**


Figure 5 presents MNLI Dev accuracy after finetuning from a checkpoint that has been pre-trained
for k steps. This allows us to answer the following
questions:

1. Question: Does BERT really need such
a large amount of pre-training (128,000
words/batch * 1,000,000 steps) to achieve
high fine-tuning accuracy?
Answer: Yes, BERTBASE achieves almost
1.0% additional accuracy on MNLI when
trained on 1M steps compared to 500k steps.


and fine-tuning, as the [MASK] symbol never appears during the fine-tuning stage. We report the
Dev results for both MNLI and NER. For NER,
we report both fine-tuning and feature-based approaches, as we expect the mismatch will be amplified for the feature-based approach as the model
will not have the chance to adjust the representations.

Masking Rates Dev Set Results


2. Question: Does MLM pre-training converge
slower than LTR pre-training, since only 15%
of words are predicted in each batch rather
than every word?
Answer: The MLM model does converge
slightly slower than the LTR model. However, in terms of absolute accuracy the MLM
model begins to outperform the LTR model
almost immediately.

**C.2** **Ablation for Different Masking**
**Procedures**


MASK SAME RND MNLI NER
Fine-tune Fine-tune Feature-based

80% 10% 10% 84.2 95.4 94.9
100% 0% 0% 84.3 94.9 94.0
80% 0% 20% 84.1 95.2 94.6
80% 20% 0% 84.4 95.2 94.7
0% 20% 80% 83.7 94.8 94.6
0% 0% 100% 83.6 94.9 94.6


In Section 3.1, we mention that BERT uses a
mixed strategy for masking the target tokens when
pre-training with the masked language model
(MLM) objective. The following is an ablation
study to evaluate the effect of different masking
strategies.
Note that the purpose of the masking strategies
is to reduce the mismatch between pre-training

84


Table 8: Ablation over different masking strategies.

The results are presented in Table 8. In the table,
MASK means that we replace the target token with
the [MASK] symbol for MLM; SAME means that
we keep the target token as is; RND means that
we replace the target token with another random
token.
The numbers in the left part of the table represent the probabilities of the specific strategies used
during MLM pre-training (BERT uses 80%, 10%,
10%). The right part of the paper represents the
Dev set results. For the feature-based approach,
we concatenate the last 4 layers of BERT as the
features, which was shown to be the best approach
in Section 5.3.
From the table it can be seen that fine-tuning is
surprisingly robust to different masking strategies.
However, as expected, using only the MASK strategy was problematic when applying the featurebased approach to NER. Interestingly, using only
the RND strategy performs much worse than our
strategy as well.


82

80


78

76

|Col1|Col2|Col3|Col4|Col5|
|---|---|---|---|---|
||||||
||||||
||||||
||||||
|||BER|TBASE (Mas|ked LM)|
|||BER|TBASE (Left|-to-Right)|


200 400 600 800 1,000

Pre-training Steps (Thousands)


Figure 5: Ablation over number of training steps. This
shows the MNLI accuracy after fine-tuning, starting
from model parameters that have been pre-trained for
_k steps. The x-axis is the value of k._

4186


-----
