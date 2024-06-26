# 🧬 EvoPrompt

This is the implementation of the paper [Connecting Large Language Models with Evolutionary Algorithms Yields Powerful Prompt Optimizers](https://arxiv.org/abs/2309.08532)

## 📃 Abstract

Large Language Models (LLMs) excel in various tasks, but they rely on carefully crafted prompts that often demand substantial human effort. To automate this process, in this paper, we propose a novel framework for discrete prompt optimization, called EvoPrompt, which borrows the idea of evolutionary algorithms (EAs) as they exhibit good performance and fast convergence. To enable EAs to work on discrete prompts, which are natural language expressions that need to be coherent and human-readable, we connect LLMs with EAs. This approach allows us to simultaneously leverage the powerful language processing capabilities of LLMs and the efficient optimization performance of EAs. Specifically, abstaining from any gradients or parameters, EvoPrompt starts from a population of prompts and iteratively generates new prompts with LLMs based on the evolutionary operators, improving the population based on the development set. We optimize prompts for both closed- and open-source LLMs including GPT-3.5 and Alpaca, on 31 datasets covering language understanding, generation tasks, as well as BIG-Bench Hard (BBH) tasks. EvoPrompt significantly outperforms human-engineered prompts and existing methods for automatic prompt generation (e.g., up to 25% on BBH). Furthermore, EvoPrompt demonstrates that connecting LLMs with EAs creates synergies, which could inspire further research on the combination of LLMs and conventional algorithms.

## 🚀 Quick Start

### ⚙️ Preparation

1. **Environmental** settings: `pip install -r requirements.txt`
2. **Data** download: The test data for the language understanding task can be found [here](https://nlp.cs.princeton.edu/projects/lm-bff/datasets.tar). Put the test file in the folder `./data/cls/{dataset_name}`
3. **OpenAI API key** required: add your OpenAI API key and other related settings in the file `auth.yaml`

### ♻ Evolution

We instanciate two evolutionary algorithms, GA (genetic algorithm) and DE (diffenrential evolution) to evolve upon the initial population. Evolve your prompts using the following commands:

Customize the parameter `--llm_type` to use `text-davinci-003`, `gpt-3.5-turbo`, `gpt-4`.

```bash
# understanding task on Alpaca
bash scripts/cls/run_ga_alpaca.sh  # Genetic algorithm
bash scripts/cls/run_de_alpaca.sh  # Differential evolution

# simplification task on Alpaca
bash scripts/sim/run_de_alpaca.sh
bash scripts/sim/run_ga_alpaca.sh

# summarization task on Alpaca
bash scripts/sum/run_de_alpaca.sh
bash scripts/sum/run_ga_alpaca.sh

# for BBH tasks
cd BBH
bash scripts/run_de_cot.sh  # DE 
bash scripts/run_ga_cot.sh  # GA
```

### 🤔 Inference

To evaluate a single instruction, run the following, set the argument `--content` to evaluate a performance of a specific prompt

```bash
bash scripts/cls/eval_single_alpaca.sh  # understanding task on alpaca
bash scripts/sim/eval_single_alpaca.sh  # simplification
bash scripts/sum/eval_single_alpaca.sh  # summarization

# BBH
cd BBH
bash scripts/eval.sh  # few-shot evaluation
```

### 📌 Notes

Note that we have two language models used in our framework, one is for evolution (argument `--llm_type`), the other for the task implementation (`--language_model`).

#### 💡Tips for Usage

The number of iteration and the population size effect the performance of EvoPrompt. There exists a trade-off between the cost and the performance. For relative simple tasks, a size of 10 and 10 iterative steps are enough, or even less. While for complex tasks, a larger population with diversity is required.

#### 🔢 Arguments

You may need to set the following arguments to customize your own configuration.

- `task`: the task category, such as `sim` (simplification), `cls`(classification), `sum`(summarization). If you need to extend this to other tasks, you may override the metric to evaluate
- `dataset`: the dataset you want to evolve prompt on
- `dev_file`: the path of the devlopment set
- `language model`: the model used for task implementation
- `llm_type`: the LLM used to evolve prompts
- `position`: this argument mainly indicates whether to use demonstration (zero-shot or few-shot)
- `sample_num`: the size of dev set, mainly used for generation task where there is no need to set the `dev_file`
- `prompt_num`: number of examples for few-shot demonstrations

## 📎 Framework

For the pipeline of EvoPrompt, there are mainly three steps as follows, while for each of them algorthms, there exists slight differences to instantiate.

- **Initialization**: We apply prompts generated manually written or generated by GPT as the initial population. (see in the `prompts.txt` and `prompts_auto.txt` under the path of each dataset)
- **Evolution** (mutation and crossover): For templates used for DE and GA, see the file `./data/templates_ga` and `./data/templates_de`. We use a demonstration including one example of the algorithm implementation to get precise and expected prompt following the steps of evolution. To avoid the LLMs copying the demonstration,the demonstration of the task is different from the task of implementation.

- **Evaluation and update**: After each iteration, we need select which prompts should be maintained in the population to update. For GA, we maintain top-$N$ prompts in each iteration while for DE, we replace the old prompt if the newly generated is better.

### 🧬 Genetic Algorithm

- **Selection strategy**: in each iteration, we need to select parents for mutation and crossover, as donors to child prompts. Set the argument `sel_mode` to apply different strategy. There are three choices: `["wheel", "random", "tour"]`, we use `wheel` by default.
- **Update**: After generating a population with the same size of the original population, $N$, we select top-$N$ as the new population.

### 🧬 Differential Evolution

- **Design in DE**: We apply different DE templates for ablations. Specify the argument `template` to use different settings.
  - Eliminate Prompt 3: `--template v2`
  - Prompt 3 (random): add the argument `--donor_random`
  - Prompt 3 (best): `--template v1` (default setting)
  - Different part: `--template v3`
- **Update**: Different from GA, in each iteration for each prompt `p`, several donor prompts are used for the new prompt `p'`, if `p'` is better than `p`, `p` will be replaced by `p'`. Otherwise, it will be maintained.

## 🌳 Code Strucutre

```python
.
├── args.py
├── auth.yaml
├── BBH  # code for BBH tasks
├── data  # dataset, templates used
│   ├── cls
│   ├── sim
│   ├── sum
│   ├── template_de.py  # templates of prompt evolution by DE
│   ├── template_ga.py  # templates of prompt evolution by GA
│   ├── template_v2.json  # templates for task implementation
│   └── templates.py  # wrapper
├── dataset.py  # dataset class
├── evaluator.py  # evaluators on different tasks
├── evoluter.py  # DE, GA, APE
├── evolution.py  # DE, GA, APE
├── get_result.py
├── infer.py  # main file for inference
├── llm_client.py  # LLM query
├── metrics.py  # metric calculation
├── requirements.txt
├── run.py  # main file for evolution
├── scripts  # scripts to run the code
└── utils.py  # auxiliary functions
```

## 🧩 Possible Extension

- **Aggregation**: Based on the final population of high quality, ensembling strategies can be effectively applied upon the prompts.
- **More fine-grained metrics**: to select prompt maintained in the population, we need to evaluate the performance on dev set. However, for understanding tasks, metrics such as accuracy or F1 are coarse-grained, sometimes it's not accurate anough to select which to keep in the population since the performances of them are the same.
- **More complex tasks** are left to explore.

## EvoPrompt's Responsible AI FAQ

### What is EvoPrompt?

* EvoPrompt generates and optimizes effective prompts with efficient performance on various tasks automatically by integrating evolutionary algorithms and LLMs. 
* Starting from initial population composed of several prompts, EvoPrompt optimizes the population iteratively by evlutionary algorithms, e.g., genetic algorithms and differential evolution algorithms.

### What can EvoPrompt do?

* EvoPrompt generates effective diecrete prompts automatically used for diverse LLMs (either open-source or black-box), such as Alpaca, ChatGPT, etc. 
* There is no need for any internal parameters or gradients and it does not  bring any training computational costs. 
* Prompts generated EvoPrompt are human-readable, interpretable and can be directly used as input for APIs. 

### What is / are EvoPrompt's intended use(s)?

- Users who call black-box LLM APIs, those who utilize ChatGPT to implement certain tasks, the performance of which is highly sensitive to the prompts or instructions. 

### What operational factors and settings allow for effective and responsible use of EvoPrompt?

- Users can set parameters such as the size of the population, the number of iterative steps since there exists a trade-off between the cost and the performance.


## ☕️ Citation

If you find this repository helpful, please consider citing our paper:

```
@article{guo2023connecting,
  title={Connecting Large Language Models with Evolutionary Algorithms Yields Powerful Prompt Optimizers},
  author={Guo, Qingyan and Wang, Rui and Guo, Junliang and Li, Bei and Song, Kaitao and Tan, Xu and Liu, Guoqing and Bian, Jiang and Yang, Yujiu},
  journal={arXiv preprint arXiv:2309.08532},
  year={2023}
}
```

## Contributing

This project welcomes contributions and suggestions. Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft
trademarks or logos is subject to and must follow
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.

## LICENSE
This project is licensed under the [MIT license](https://github.com/microsoft/EvoPrompt/blob/main/LICENSE) in the root directory of this source tree.
