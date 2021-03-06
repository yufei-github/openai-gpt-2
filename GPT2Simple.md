gpt-2-simple
gen_demo

A simple Python package that wraps existing model fine-tuning and generation scripts for OpenAI's GPT-2 text generation model (specifically the "small" 124M and "medium" 355M hyperparameter versions). Additionally, this package allows easier generation of text, generating to a file for easy curation, allowing for prefixes to force the text to start with a given phrase.

This package incorporates and makes minimal low-level changes to:

Model management from OpenAI's official GPT-2 repo (MIT License)
Model finetuning from Neil Shepperd's fork of GPT-2 (MIT License)
Text generation output management from textgenrnn (MIT License / also created by me)
For finetuning, it is strongly recommended to use a GPU, although you can generate using a CPU (albeit much more slowly). If you are training in the cloud, using a Colaboratory notebook or a Google Compute Engine VM w/ the TensorFlow Deep Learning image is strongly recommended. (as the GPT-2 model is hosted on GCP)

You can use gpt-2-simple to retrain a model using a GPU for free in this Colaboratory notebook, which also demos additional features of the package.

Install
gpt-2-simple can be installed via PyPI:

pip3 install gpt-2-simple
You will also need to install the corresponding TensorFlow for your system (e.g. tensorflow or tensorflow-gpu). TensorFlow 2.0 is currently not supported and the package will throw an assertion if loaded, so TensorFlow 1.14/1.15 is recommended.

Usage
An example for downloading the model to the local system, finetuning it on a dataset. and generating some text.

Warning: the pretrained 124M model, and thus any finetuned model, is 500 MB! (the pretrained 355M model is 1.5 GB)

import gpt_2_simple as gpt2
import os
import requests

model_name = "124M"
if not os.path.isdir(os.path.join("models", model_name)):
	print(f"Downloading {model_name} model...")
	gpt2.download_gpt2(model_name=model_name)   # model is saved into current directory under /models/124M/


file_name = "shakespeare.txt"
if not os.path.isfile(file_name):
	url = "https://raw.githubusercontent.com/karpathy/char-rnn/master/data/tinyshakespeare/input.txt"
	data = requests.get(url)
	
	with open(file_name, 'w') as f:
		f.write(data.text)
    

sess = gpt2.start_tf_sess()
gpt2.finetune(sess,
              file_name,
              model_name=model_name,
              steps=1000)   # steps is max number of training steps

gpt2.generate(sess)
The generated model checkpoints are by default in /checkpoint/run1. If you want to load a model from that folder and generate text from it:

import gpt_2_simple as gpt2

sess = gpt2.start_tf_sess()
gpt2.load_gpt2(sess)

gpt2.generate(sess)
As with textgenrnn, you can generate and save text for later use (e.g. an API or a bot) by using the return_as_list parameter.

single_text = gpt2.generate(sess, return_as_list=True)[0]
print(single_text)
You can pass a run_name parameter to finetune and load_gpt2 if you want to store/load multiple models in a checkpoint folder.

There is also a command-line interface for both finetuning and generation with strong defaults for just running on a Cloud VM w/ GPU. For finetuning (which will also download the model if not present):

gpt_2_simple finetune shakespeare.txt
And for generation, which generates texts to files in a gen folder:

gpt_2_simple generate
Most of the same parameters available in the functions are available as CLI arguments, e.g.:

gpt_2_simple generate --temperature 1.0 --nsamples 20 --batch_size 20 --length 50 --prefix "<|startoftext|>" --truncate "<|endoftext|>" --include_prefix False --nfiles 5
See below to see what some of the CLI arguments do.

NB: Restart the Python session first if you want to finetune on another dataset or load another model.

Differences Between gpt-2-simple And Other Text Generation Utilities
The method GPT-2 uses to generate text is slightly different than those like other packages like textgenrnn (specifically, generating the full text sequence purely in the GPU and decoding it later), which cannot easily be fixed without hacking the underlying model code. As a result:

In general, GPT-2 is better at maintaining context over its entire generation length, making it good for generating conversational text. The text is also generally gramatically correct, with proper capitalization and few typoes.
The original GPT-2 model was trained on a very large variety of sources, allowing the model to incorporate idioms not seen in the input text.
GPT-2 can only generate a maximum of 1024 tokens per request (about 3-4 paragraphs of English text).
GPT-2 cannot stop early upon reaching a specific end token. (workaround: pass the truncate parameter to a generate function to only collect text until a specified end token. You may want to reduce length appropriately.)
Higher temperatures work better (e.g. 0.7 - 1.0) to generate more interesting text, while other frameworks work better between 0.2 - 0.5.
When finetuning GPT-2, it has no sense of the beginning or end of a document within a larger text. You'll need to use a bespoke character sequence to indicate the beginning and end of a document. Then while generating, you can specify a prefix targeting the beginning token sequences, and a truncate targeting the end token sequence. You can also set include_prefix=False to discard the prefix token while generating (e.g. if it's something unwanted like <|startoftext|>).
If you pass a single-column .csv file to finetune(), it will automatically parse the CSV into a format ideal for training with GPT-2 (including prepending <|startoftext|> and suffixing <|endoftext|> to every text document, so the truncate tricks above are helpful when generating output). This is necessary to handle both quotes and newlines in each text document correctly.
GPT-2 allows you to generate texts in parallel by setting a batch_size that is divisible into nsamples, resulting in much faster generation. Works very well with a GPU (can set batch_size up to 20 on Colaboratory's K80)!
Due to GPT-2's architecture, it scales up nicely with more powerful GPUs. For the 124M model, if you want to train for longer periods of time, GCP's P100 GPU is about 3x faster than a K80/T4 for only 3x the price, making it price-comparable (the V100 is about 1.5x faster than the P100 but about 2x the price). The P100 uses 100% of the GPU even with batch_size=1, and about 88% of the V100 GPU.
If you have a partially-trained GPT-2 model and want to continue finetuning it, you can set overwrite=True to finetune, which will continue training and remove the previous iteration of the model without creating a duplicate copy. This can be especially useful for transfer learning (e.g. heavily finetune GPT-2 on one dataset, then finetune on other dataset to get a "merging" of both datasets).
If your input text dataset is massive (>100 MB), you may want to preencode and compress the dataset using gpt2.encode_dataset(file_path). THe output is a compressed .npz file which will load much faster into the GPU for finetuning.
The 774M "large" model may support finetuning because it will cause modern GPUs to go out-of-memory (you may get lucky if you use a P100 GPU on Colaboratory). However, you can still generate from the default pretrained model using gpt2.load_gpt2(sess, model_name='774M') and gpt2.generate(sess, model_name='774M').
The 1558M "extra large", true model, may not work out-of-the-box with the GPU included with the Colaboratory Notebook. More testing is needed to identify optimial configurations for it.
Interactive Apps Using gpt-2-simple
gpt2-small — App using the default GPT-2 124M pretrained model
gpt2-reddit — App to generate Reddit titles based on a specified subreddit and/or keyword(s)
gpt2-mtg — App to generate Magic: The Gathering cards
Text Generation Examples Using gpt-2-simple
ResetEra — Generated video game forum discussions (GitHub w/ dumps)
/r/legaladvice — Title generation (GitHub w/ dumps)
Hacker News — Tens of thousands of generated Hacker News submission titles
Maintainer/Creator
Max Woolf (@minimaxir)

Max's open-source projects are supported by his Patreon. If you found this project helpful, any monetary contributions to the Patreon are appreciated and will be put to good creative use.

License
MIT

Disclaimer
This repo has no affiliation or relationship with OpenAI.

gpt-2-simple的Python项目详细描述
模型文本asstepssimplesessopenai微调gpt2gpt
一个简单的Python包，它封装了OpenAI GPT-2文本生成模型（特别是“小”，124M超参数版本）的现有模型微调和生成脚本。此外，这个包允许更容易地生成文本，生成文件以便于管理，允许前缀强制文本以给定短语开头。

用法
一个将模型下载到本地系统的示例，可以在数据集上精细地打开它。生成一些文本。

警告：预先训练的模型，因此任何微调的模型，是500兆字节！

importgpt_2_simpleasgpt2gpt2.download_gpt2()# model is saved into current directory under /models/124M/sess=gpt2.start_tf_sess()gpt2.finetune(sess,'shakespeare.txt',steps=1000)# steps is max number of training stepsgpt2.generate(sess)
默认情况下，生成的模型检查点位于/checkpoint/run1中。如果要从该文件夹加载模型并从中生成文本：

importgpt_2_simpleasgpt2sess=gpt2.start_tf_sess()gpt2.load_gpt2(sess)gpt2.generate(sess)
与textgenrnn一样，您可以使用return_as_list参数生成和保存文本以供以后使用（例如api或bot）。

single_text=gpt2.generate(sess,return_as_list=True)[0]print(single_text)
如果要在checkpoint文件夹中存储/加载多个模型，可以将run_name参数传递给finetune和load_gpt2

注意：如果要在另一个数据集上进行微调或加载另一个模型，请先重新启动python会话。

该 Python 包包含以下内容，并对其进行了最小程度的低级更改：

来自 OpenAI 官方 GPT-2 库的模型管理（MIT 许可证）
来自 GPT-2 中 Neil Shepperd fork 的模型微调（MIT 许可证）
来自 textgenrnn 的文本生成输出管理（MIT 许可证）
为了微调，该项目强烈建议你使用 GPU，虽然你用 CPU 也可以生成（但速度会慢很多）。如果你在云端训练，强烈建议你使用 Colaboratory notebook 或带有 TensorFlow 深度学习图像的谷歌计算引擎 VM（因为 GPT-2 模型位于 GCP 上）。

你可以使用 gpt-2-simple 在这个 Colaboratory notebook 中免费用 GPU 来重新训练模型，该 notebook 还演示了这个软件包的其它功能。

Colaboratory notebook 地址：https://colab.research.google.com/drive/1VLG8e7YSEwypxU-noRNhsv5dW4NfTGce

安装

gpt-2-simple 可以通过 PyPI 来安装：

pip3 install gpt_2_simple
你还要为你的系统安装相应的 TensorFlow（如 tensorflow 或 tensorflow-gpu）

使用

将模型下载到本地系统的示例，在数据集上对它进行微调，然后生成一些文本。

警告：模型是预训练的，因此任何微调模型都是 500MB。

import gpt_2_simple as gpt2

gpt2.download_gpt2()   # model is saved into current directory under /models/117M/

sess = gpt2.start_tf_sess()
gpt2.finetune(sess, 'shakespeare.txt', steps=1000)   # steps is max number of training steps

gpt2.generate(sess)
生成模型的检查点默认在/checkpoint/run1 中。如果你想从该文件夹中加载模型并从中生成文本：

import gpt_2_simple as gpt2

sess = gpt2.start_tf_sess()
gpt2.load_gpt2(sess)

gpt2.generate(sess)
与 textgenrnn 一样，你可以用 return_as_list 参数生成并保存文本供以后使用（如 API 或机器人）。

single_text = gpt2.generate(sess, return_as_list=True)[0]
print(single_text)
如果你想在 checkpoint 文件夹中存储或加载多个模型，可以把 run_name 参数传递给 finetune 和 load_gpt2。

注意：如果你想在另一个数据集上进行微调或加载另一个模型，先重启 Python 会话。

gpt-2-simple 和其它文本生成程序的区别

GPT-2 用来生成文本的方法与 textgenrnn 等其它安装包（特别是纯粹使用 GPU 生成完整文本序列并随后对其进行解码的安装包）使用的方法略有不同，这些方法在没有破解底层模型代码的情况下无法轻易修复。

所以：

一般来说，GPT-2 更擅长在整个生成长度上维护上下文，从而能够有效地生成对话文本。文本在语法上通常也是正确的，并且有适当的大写和较少的打印错误。
原始 GPT-2 模型在大量来源的文本上进行训练，使该模型包含输入文本中看不到的趋势。
GPT-2 针对每个请求最多只能生成 1024 个 token（约是 3-4 段英语文本）。
GPT-2 在到达特定的结束 token 时无法提前停止。（暂时解决方法：将 truncate 参数传递给 generate 函数，以便只收集文本，直至到达特定的结束 token。你可能想适当地缩小 length。）
较高温度（如 0.7-1.0）能够更好地生成更有趣的文本，而其它框架在温度 0.2-0.5 之间运转更好。
当对 GPT-2 进行微调时，它并不清楚较大文本中文档的开头或结尾。你需要使用定制的字符序列来显示文档的开头或结尾。之后在文本生成中，你可以指定针对开始 token 序列的 prefix 和针对结束 token 序列的 truncate。
通过设置一个可分成 nsamples 的 batch_size，你可以使用 GPT-2 生成并行文本，从而加快生成速度。GPT-2 与 GPU 配合得很好（可以在 Colaboratory K80 上将 batch_size 设置为 20）！
计划工作

注意：除非需求另有规定，否则本项目的范围非常小。

允许用户生成超过 1024 个 token 的文本。
允许用户使用 Colaboratory 的 TPU 进行微调。
允许用户使用多个 GPU（如 Horovod）。
对于 Colaboratory，允许模型在训练期间自动将检查点保存至 Google Drive，以防止超时。
使用 gpt-2-simple 的示例

ResetEra：生成视频游戏论坛讨论

地址：https://www.resetera.com/threads/i-trained-an-ai-on-thousands-of-resetera-thread-conversations-and-it-created-hot-gaming-shitposts.112167/

项目创建者：Max Woolf

基于 GPT-2 的「故事生成器」

GPT-2 强大的模型不仅吸引了众多机器学习从业者的关注，其「脑补」故事的能力也让人们不禁有了很多大胆的想法。为了让更多人能够接触最新技术，另一个开发者 eukaryote 最近还推出了一个新网站：This Story Does Not Exist

链接：https://www.thisstorydoesnotexist.com/

这是一个基于 GPT-2 的文本生成器。在这里，每个人都可以输入一段文字，看看人工智能会给你讲一段什么样的故事
