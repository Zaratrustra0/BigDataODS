# 利用GPT-3 Fine-tunes训练专属语言模型
![](https://img-blog.csdnimg.cn/cafddabf2d2a470ea839dc540c9a7565.jpeg#pic_center)
  

什么是模型[微调](https://so.csdn.net/so/search?q=%E5%BE%AE%E8%B0%83&spm=1001.2101.3001.7020)（fine-tuning）？
---------------------------------------------------------------------------------------------------

ChatGPT已经使用来自互联网的海量开放数据进行了[预训练](https://so.csdn.net/so/search?q=%E9%A2%84%E8%AE%AD%E7%BB%83&spm=1001.2101.3001.7020)，对于任何输入都可以给出通用回答。如果我们想让ChatGPT的回答更有针对性，我们可以在输入时给出示例，ChatGPT可以通过“示例学习”（few-shot learning）理解你希望它完成的任务，并产生类似的合理输出。

但是“示例学习”每次需要给出示例，使用起来很不方便。微调（[fine-tuning](https://so.csdn.net/so/search?q=fine-tuning&spm=1001.2101.3001.7020)）可以通过训练更多示例来改进“短学习”，使用微调后的模型将不再需要在输入中提供示例。 这样既可以节约成本又可以实现更低延迟的请求。

更重要的是，对于一些专业场景，预训练模型可能无法达到理想的输出效果。此时需要我们给出更加具体和对口的数据对模型进行专门的强化，使其能够更好地回答该领域的问题，从而提升整体效果。

简而言之，微调允许我们将自定义数据集与大型语言模型（LLM）相匹配，让模型在我们的特定任务场景下依然表现良好。

为什么需要模型微调？
----------

### 微调 vs 重新训练

这里有必要区分一下_微调（fine-tuning）_ 和 _重新训练（re-training）_ 的概念。

简单地说，重新训练是用新数据从头开始训练模型，而微调是用新数据调整先前训练模型的参数。

针对特定任务场景，**微调比重新训练无论从时间还是费用上都更快更经济**。

从头重新训练GPT-3 或 ChatGPT成本高的惊人。据估算，GPT-3一次训练的成本大约140万美元，ChatGPT模型更大，一次训练大约需要1200万美元。这还不包括上万颗A100GPU的成本。一块Nvidia A100 80G显存显卡按5万计算，1万块A100显卡光初始费用就5个小目标！很少有企业能够负担得起这么巨大的软硬件支出。

在GPT-3上微调也有成本，以功能最强的Davinci为例，训练成本0.03美元/1千token，这个成本相比重新训练天壤之别。下图是微调模型的训练和使用成本：

![](https://img-blog.csdnimg.cn/aa7a3c671e84490bb94dcbddc6d14fe0.png#pic_center)

所以目前对大多数企业来说，只适合在GPT-3上做微调，除了少数巨头，绝大多数企业都没实力和能力重新训练。

### 微调 vs 提示设计

GPT-3支持“示例学习”（few-shot learning），我们可以通过在输入prompt时给予示例来提升模型输出效果，但提升效果远不如微调的效果，下面是微调和提示设计的效果对比：

![](https://img-blog.csdnimg.cn/603869a5397142669835bd7fb69eab94.png#pic_center)

对比提示设计，微调模型可以获得如下优势：

*   更好的输出效果
*   比示例学习更多的训练数据
*   减少token消耗从而节约成本
*   请求延迟更低

训练专属模型
------

可以通过以下6个主要步骤开始创建微调模型。为了方便大家理解，我会结合我们在GPT-3上定制客服机器人的Python代码来演示微调过程。

![](https://img-blog.csdnimg.cn/ceb63d8e62c54459910e6014aaea5631.png#pic_center)

### 数据准备

GPT-3微调需要的数据格式是专属JSONL格式，形式如下：

```json
{"prompt": "<prompt text>", "completion": "<ideal generated text>"}
{"prompt": "<prompt text>", "completion": "<ideal generated text>"}
{"prompt": "<prompt text>", "completion": "<ideal generated text>"}

```

上面的数据格式很好理解——每行都包含一个`prompt`和一个`completion`，表示特定提示对应的理想文本。

我们日常系统中的数据一般不会保存为JSONL格式，因此需要先将数据转换为JSONL格式。OpenAI提供了命令行工具来帮我们将常见数据格式转化为JSONL格式，用法如下：

```bash
openai tools fine_tunes.prepare_data -f <LOCAL_FILE>

```

其中`<LOCAL_FILE>`传入本地数据文件，支持**CSV、TSV、XLSX、JSON** 和 **JSONL**格式，只要文件中的数据格式是包含 `prompt` 和 `completion` 列或关键字就行。

数据准备阶段大家经常问的一个问题是*“我要准备多少数据用于微调才够？”*。通常来说，自然是“多多益善”，但由于微调设计训练成本，我们需要从中取得一个平衡。Open AI建议至少提供150–200个微调示例，但我个人在实际项目中发现150-200个微调示例往往不够，建议先**从几百到一千条数据**作为起始测试，根据微调后的模型效果再决定是否追加更多训练数据。

![](https://img-blog.csdnimg.cn/5159658c74d94ce087f54767fbbb6279.png#pic_center)

> GPT-3支持自定义模型的持续微调，因此你可以随时用新数据在之前微调的模型上做进一步微调。

### 清洗数据

相比数据数量，**数据质量更为关键**。

GPT-3本质上是一个大型神经网络，它对我们来说是一个黑盒。所以它是典型的“garbage in, garbage out”，模型输出质量与训练数据质量有着直接的关系。

数据质量和多样性越高，模型就会工作得越好。通常需要一组不同的示例，以确保模型能够很好地泛化到新的示例。最好能够提供一个正面示例和负面示例，以确保模型能够处理各种输入。

为了验证微调模型质量，我们通常会将数拆分为训练集和验证集，通常按照 80 % / 20 % 80\\%/20\\% 80%/20%的比例进行拆分。

> ⚠注意：除JSONL格式外，训练数据和验证数据文件必须是UTF-8编码，并包含字节顺序标记（BOM），文件大小不能超过200MB。

### 构建模型

微调数据准备好后，我们就协议开始着手微调模型了。在开始微调训练之前，我们需要先确定微调的基础模型。

每个微调工作都从一个基础模型开始，默认为`curie`。基础模型不同会影响模型的性能和微调模型的成本。目前支持微调的基础模型包括：`ada`、`babbage`、`curie`或`davinci`。

下面是用Python完成模型构建：

```python
import openai

openai.api_key = "YOUR_API_KEY"

resp = openai.FineTune.create(training_file="training_file_path", 			
                              validation_file="validation_file_path", 
                              check_if_files_exist=True, 
                              model="davinci")
job_id = resp["id"]
status = resp["status"]
print(f'微调任务ID: {job_id}，状态: {status}\n')

```

上面的代码选择`davinci`作为基础模型，并传入了训练集和验证集的本地文件路径，创建微调任务。如果创建成功，则会返回微调任务的id及状态。

创建微调任务还支持其他参数，说明如下：

| 参数名 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `training_file` | string |  | 训练集文件路径，必须是JSONL格式 |
| `validation_file` | string | `null` | 验证集文件路径，必须是JSONL格式  
如果提供，则在微调期间，定期生成验证度量。这些指标可以在微调结果文件中查看。  
训练集数据与验证集数据必须互斥。 |
| `check_if_files_exist` | boolean | `true` | 是否检验文件是否存在 |
| `model` | string | `curie` | 要微调的基础模型的名称。可以选择`ada`、`babbage`、`curie`、`davinci`或2022-04-21之后创建的微调模型。 |
| `n_epochs` | int | `4` | 训练几轮。 |
| `batch_size` | int | `null` | 训练的批次大小。默认情况下，批次大小将动态配置为训练集样本数的约0.2%，上限为256。通常来说较大的批次大小对于较大的数据集更有效。 |
| `learning_rate_multiplier` | float | `null` | 学习率系数。微调学习率=预训练的原始学习率乘以该值。  
默认情况下，学习率系数为0.05、0.1或0.2，具体取决于最终批次大小（较大的学习率往往在较大的批次大小下表现更好）。我们建议使用0.02到0.2范围内的值进行实验，以查看产生最佳结果的方法。 |
| `prompt_loss_weight` | float | `0.01` | 提示损失权重。控制模型学习生成提示（生成输出的权重为1），并增加输出较短时训练的稳定性。  
如果提示非常长（相对于输出），那么减少这个权重可以避免过度学习。 |
| `compute_classification_metrics` | boolean | `false` | 如果为true，则在每轮训练结束时使用验证集计算效果，例如准确度和F-1 score 。这些指标可以在结果文件中查看。 |
| `classification_n_classes` | int | `null` | 分类任务中的类别数。  
多类别分类需要此参数。 |
| `classification_positive_class` | string | `null` | 二分类任务中的正例。  
在进行二分类时，需要此参数来生成精度、召回率和F1 score 。 |
| `classification_betas` | array | `null` | 如果提供，将按照指定的beta值计算F-beta分数。F-beta是F-1的概括。仅用于二分类任务。  
beta为1（即F-1分数）时，准确率和召回率的权重相同。beta越大，召回率的权重越大，准确率的权重越小。beta分数越小，准确率权重越高，召回率权重越低。 |
| `suffix` | string | `null` | 最多40个字符的字符串，将添加到微调后的模型名称中。 |

### 微调模型

微调任务创建之后通常处于**Pending**状态，这是因为OpenAI系统中通常有其他任务排在你之前，我们的任务会先放在队列中，等待被处理。一般来说一旦进入训练状态，微调训练可能需要几分钟或几小时，具体取决于选择的基础模型和数据集的大小。我们可以用微调任务id来查询微调任务的状态：

```python
while status not in ["succeeded", "failed"]:
    time.sleep(2)
    
    status = openai.FineTune.retrieve(id=job_id)["status"]
    print(f'微调任务ID: {job_id}，状态: {status}')
    
print(f'微调任务ID: {job_id} 完成， 结束状态: {status}\n')

```

### 评估模型

微调成功后都会有训练结果输出，可以通过如下代码获取评估结果：

```python
fine_tune = openai.FineTune.retrieve(id=job_id)
result_files = fine_tune.get("result_files", [])
if result_files:
    result_file = result_files[0]
    resp = openai.File.download(id=result_file["id"])
    print(resp.decode("utf-8"))

```

这里有丰富的模型评估数据，供我们对模型微调质量进行评估。

### 部署模型

如果模型效果满意，我们就可以将模型投入生产使用了。`openai.FineTune.retrieve()`方法返回的数据结构中的`fine_tuned_model`就是微调后的模型名称。可以直接拿这个模型名称在API中使用。

```python
model_name = openai.FineTune.retrieve(id=job_id)["fine_tuned_model"]

response = openai.Completion.create(
  model=model_name,
  prompt="今天晚上吃什么好呢？\n",
  temperature=0.7,
  max_tokens=256,
  top_p=1,
  frequency_penalty=0,
  presence_penalty=0,
  stop=["END"]
)

```

总结
--

ChatGPT最让人惊艳的一点，就在于能够像人一样去对话。这种流畅的人机对话背后所展现的强大的自然语言理解力和表达力，目前只表现在通用领域。一旦进入某个专业领域，ChatGPT经常会“一本正经，胡说八道”。此时用特定领域的知识对模型进行微调是时间成本和经济成本最高的解决方案。事实证明，哪怕是最小的训练数据，也会带来明显的表现提升。

未来随着LLM变得更大、更易访问和开源，相信在不久的将来，我们可以看到微调在自然语言处理中无处不在。同时我也非常期待边缘学习的突破，可以将大模型训练的成本降下来，那时我们再来看如何从头重新训练一个大模型。