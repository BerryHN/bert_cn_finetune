## BERT下游任务finetune列表

finetune基于官方代码改造的模型基于pytorch/tensorflow双版本

*** 2019-10-15: 增加tensorflow(bert/roberta)在cmrc2018上的finetune代码 ***

2019-10-14: 新增DRCD test结果

*** 2019-10-12: pytorch支持albert ***

### 模型及相关代码来源

1. 官方Bert (https://github.com/google-research/bert)

2. transformers (https://github.com/huggingface/transformers)

3. 哈工大讯飞预训练 (https://github.com/ymcui/Chinese-BERT-wwm)

4. brightmart预训练 (https://github.com/brightmart/roberta_zh)

5. 自己瞎折腾的siBert (https://github.com/ewrfcas/SiBert_tensorflow)

### 关于pytorch的FP16

FP16的训练可以显著降低显存压力(如果有V100等GPU资源还能提高速度)。但是最新版编译的apex-FP16对并行的支持并不友好(https://github.com/NVIDIA/apex/issues/227)。  
实践下来bert相关任务的finetune任务对fp16的数值压力是比较小的，因此可以更多的以计算精度换取效率，所以我还是倾向于使用老版的FusedAdam+FP16_Optimizer的组合。  
由于最新的apex已经舍弃这2个方法了，需要在编译apex的时候额外加入命令--deprecated_fused_adam  
```
pip install -v --no-cache-dir --global-option="--cpp_ext" --global-option="--cuda_ext"  --global-option="--deprecated_fused_adam" ./
```

### 关于tensorflow的blocksparse

blocksparse(https://github.com/openai/blocksparse)可以在tensorflow1.13版本直接pip安装，否则可以自己clone后编译。  
其中fast_gelu以及self-attention中的softmax能够极大缓解显存压力。另外部分dropout位置我有所调整，整体显存占用下降大约30%~40%。

tensorflow roberta_large length=512 fp16

model | length | batch | memory |
| ------ | ------ | ------ | ------ |
| roberta_base | 512 | 32 | 16GB |
| roberta_large | 512 | 12 | 16GB |


### 参与任务

1. CMRC 2018：篇章片段抽取型阅读理解（简体中文，只测了dev）

2. DRCD：篇章片段抽取型阅读理解（繁体中文，转简体, 只测了dev）

3. CJRC: 法律阅读理解（简体中文, 只有训练集，统一90%训练，10%测试）

4. XNLI：自然语言推断 (todo)

5. Msra-ner：中文命名实体识别 (todo)

6. THUCNews：篇章级文本分类 (todo)

7. Dbqa: ...

8. Chnsenticorp: ...

9. Lcqmc: ...

### 评测标准

验证集一般会调整learning_rate，warmup_rate，train_epoch等参数，选择最优的参数用五个不同的随机种子测试5次取平均和括号内最大值。测试集会直接用最佳的验证集模型进行验证。

### 模型介绍

L(transformer layers), H(hidden size), A(attention head numbers), E(embedding size)

**特别注意brightmart roberta_large所支持的max_len只有256**

| models | config |
| ------ | ------ |
| siBert_base | L=12, H=768, A=12, max_len=512 |
| siALBert_middle | L=16, H=1024, E=128, A=16, max_len=512 |
| 哈工大讯飞 roberta_wwm_ext_base | L=12, H=768, A=12, max_len=512 |
| brightmart roberta_middle | L=24, H=768, A=12, max_len=512 |
| brightmart roberta_large | L=24, H=1024, A=16, **max_len=256** |


### 结果

#### 参数

| models | cmrc2018 | DRCD | CJRC |
| ------ | ------ | ------ | ------ |
| 哈工大讯飞 roberta_wwm_ext_base | epoch2, batch=32, lr=3e-5, warmup=0.1 | 同左 | 同左 |
| brightmart roberta_middle | epoch2, batch=32, lr=3e-5, warmup=0.1 | 同左 | 同左 |
| brightmart roberta_large | epoch2, batch=32, lr=3e-5, warmup=0.1 | 同左 | 同左 |
| brightmart albert_large |  epoch3, batch=32, lr=2e-5, warmup=0.05 | epoch3, batch=32, lr=2e-5, warmup=0.05 | epoch2, batch=32, lr=3e-5, warmup=0.1 |

#### cmrc2018(阅读理解)

| models | DEV |
| ------ | ------ |
| sibert_base | F1:87.521(88.628) EM:67.381(69.152) |
| sialbert_middle | F1:87.6956(87.878) EM:67.897(68.624) |
| 哈工大讯飞 roberta_wwm_ext_base | F1:87.521(88.628) EM:67.381(69.152) |
| brightmart roberta_middle | F1:86.841(87.242) EM:67.195(68.313) |
| brightmart roberta_large | **F1:88.608(89.431) EM:69.935(72.538)** |
| brightmart albert_large | F1:87.8596(88.43) EM:67.754(69.028) |

#### DRCD(阅读理解)

| models | DEV | TEST |
| ------ | ------ | ------ |
| siBert_base | F1:93.343(93.524) EM:87.968(88.28) | F1:92.818 EM:86.745 |
| siALBert_middle | F1:93.865(93.975) EM:88.723(88.961) | F1:93.857 EM:88.033 |
| 哈工大讯飞 roberta_wwm_ext_base | F1:94.257(94.48) EM:89.291(89.642) | F1:93.526 EM:88.119 |
| brightmart roberta_large | **F1:94.933(95.057) EM:90.113(90.238)** | **F1:94.254 EM:89.350** |
| brightmart albert_large | F1:93.9034(94.034) EM:88.882(89.132) | F1:93.057 EM:87.518 |

#### CJRC(带有yes,no,unkown的阅读理解)

| models | DEV |
| ------ | ------ |
| siBert_base | F1:80.714(81.14) EM:64.44(65.04) |
| siALBert_middle | F1:80.9838(81.299) EM:63.796(64.202) |
| 哈工大讯飞 roberta_wwm_ext_base | F1:81.510(81.684) EM:64.924(65.574) |
| brightmart roberta_large | F1:80.16(80.475) EM:65.249(66.133) |
| brightmart albert_large | F1:81.113(81.563) EM:65.346(65.727) |


