***

# Modern LLM Engineering – Deep-Dive Concepts

***

## 1. SLMs vs LLMs

### 1.1 What is a language model?

A language model estimates the probability of a sequence of tokens, typically via next-token prediction. It learns patterns in text and can then generate or understand language.

### 1.2 Large Language Models (LLMs)

- Very high parameter counts: typically tens to hundreds of billions of parameters and beyond. [reddit](https://www.reddit.com/r/LargeLanguageModels/comments/1ifia7u/can_someone_please_explain_to_me_what_is_the/)
- Trained on massive, diverse corpora (internet-scale text, code, etc.) with huge compute budgets. [splunk](https://www.splunk.com/en_us/blog/learn/foundation-models.html)
- Strong generalization across many tasks “out of the box”: reasoning, coding, summarization, translation, etc. [geeksforgeeks](https://www.geeksforgeeks.org/artificial-intelligence/llms-vs-slms-comparative-analysis-of-language-model-architectures/)
- High training and inference cost; usually deployed on powerful cloud GPUs or specialized accelerators. [abbyy](https://www.abbyy.com/blog/small-vs-large-language-models/)

Interview framing:  
LLMs are general-purpose “foundation brains” with broad world knowledge, but they are heavy and expensive to run.

### 1.3 Small Language Models (SLMs)

- Much fewer parameters: from a few hundred million to a few billion; often < 10–30B. [sendbird](https://sendbird.com/blog/small-language-models)
- Often focus on narrower domains or tasks (support, search, specific verticals). [redhat](https://www.redhat.com/en/topics/ai/llm-vs-slm)
- Easier and cheaper to train, fine-tune, and deploy; fit on smaller GPUs, can run on edge devices or mobile. [redhat](https://www.redhat.com/en/topics/ai/llm-vs-slm)
- Can outperform massive LLMs on well-scoped tasks when they are highly specialized. [geeksforgeeks](https://www.geeksforgeeks.org/artificial-intelligence/llms-vs-slms-comparative-analysis-of-language-model-architectures/)

Interview soundbite:  
“LLMs give breadth and general reasoning; SLMs give efficiency and focus. In a product, I might use an SLM on-device for quick, private tasks and call an LLM in the cloud for complex reasoning.”

### 1.4 When to use which?

- Use LLMs when:
  - You need strong general reasoning, broad domain coverage, or you’re building a frontier research system.
- Use SLMs when:
  - Latency, cost, privacy, or on-device execution are critical.
  - The problem is narrow enough that a smaller, specialized model works well. [abbyy](https://www.abbyy.com/blog/small-vs-large-language-models/)

***

## 2. Foundation Model Training

### 2.1 What is a foundation model?

A foundation model is a large, general-purpose model trained on huge, diverse datasets that can be adapted to many downstream tasks. [aws.amazon](https://aws.amazon.com/what-is/foundation-models/)

Properties:

- Trained once (expensively) and reused many times.
- Acts as a base for fine-tuning, prompting, adapters, etc.
- Supports transfer learning across many tasks.

### 2.2 Training objective and paradigm

Most language foundation models use self-supervised learning:

- Task: predict the next token (or masked token) given context. [ibm](https://www.ibm.com/think/topics/self-supervised-learning)
- Data: raw text without manual labels.
- The “labels” are derived from the input itself (the next word is the label). [coursera](https://www.coursera.org/articles/foundation-model)

Why self-supervised:

- Scales to trillions of tokens without manual annotation. [splunk](https://www.splunk.com/en_us/blog/learn/foundation-models.html)
- Learns rich representations of language that transfer well to many tasks. [coursera](https://www.coursera.org/articles/foundation-model)

Interview phrasing:  
“A foundation model is the ‘pretrained brain’ learned via self-supervised objectives on massive data. Everything else—fine-tuning, RLHF, adapters—is about specializing or aligning that brain.”

***

## 3. Supervised Fine-Tuning (SFT)

### 3.1 What is SFT?

Supervised fine-tuning adjusts a pretrained model using labeled examples of input → desired output.

Examples:

- Instruction tuning: `(prompt, ideal response)` pairs.
- Task tuning: translation, summarization, classification with ground-truth outputs.

### 3.2 Role in modern LLM pipelines

Typical pipeline:

1. Train foundation model with self-supervision.
2. Apply SFT on curated instruction datasets to make the model follow instructions more reliably.
3. Optionally, further align via RLHF or related methods.

What SFT does:

- Teaches the model to map from “user-style instructions” to “assistant-style responses”.
- Reduces incoherent or off-task answers by showing many examples of desired behavior.

Interview line:  
“SFT is where you turn a raw foundation model that’s good at next-token prediction into an instruction-following assistant tailored to a set of tasks.”

***

## 4. RLHF (Reinforcement Learning from Human Feedback)

### 4.1 Why RLHF?

Even after SFT, the model may:

- Give harmful or unsafe outputs.
- Be verbose, evasive, or not aligned with product values.
- Prefer some styles of response over others.

Human preferences are richer than simple right/wrong labels, so RLHF is used to encode them into the model behavior.

### 4.2 High-level RLHF pipeline

Standard (simplified) three-step pipeline:

1. **Supervised fine-tuning (SFT)**  
   - Train on prompt → “good” response pairs created by humans.
2. **Reward model training**  
   - Collect multiple candidate responses per prompt.
   - Humans rank or rate them (e.g., best to worst).
   - Train a reward model to predict which response humans prefer.
3. **RL optimization**  
   - Use a reinforcement learning algorithm (e.g., PPO) to further fine-tune the base model.
   - The objective is to maximize the reward given by the reward model while constraining deviation from the original model.

### 4.3 What does RLHF achieve?

- Pushes the model towards outputs that humans prefer (helpful, harmless, honest).
- Encodes product constraints like safety policies or style guidelines.
- Balances alignment with preserving capabilities by constraining divergence from the original model.

Interview summary:  
“RLHF is how we turn a capable but raw model into one that behaves according to human preferences and safety policies, by training a reward model from human judgments and then optimizing the language model against that reward.”

***

## 5. Fine-Tuning Techniques

### 5.1 Full fine-tuning

- Update all model parameters during task-specific training. [geeksforgeeks](https://www.geeksforgeeks.org/deep-learning/fine-tuning-using-lora-and-qlora/)
- Pros:
  - Maximum flexibility; can adapt deeply to new tasks.
  - Often gives best performance when you have lots of data and compute. [geeksforgeeks](https://www.geeksforgeeks.org/deep-learning/fine-tuning-using-lora-and-qlora/)
- Cons:
  - Extremely expensive for large models.
  - Requires large GPUs and complex training infrastructure.
  - Each task/version needs its own full copy of weights.

Useful when:

- You control the whole training stack at scale (e.g., big labs).
- You need strong performance on a specific task and can afford the cost.

### 5.2 PEFT (Parameter-Efficient Fine-Tuning)

Parameter-efficient fine-tuning updates only a small subset of parameters or adds small trainable modules. [statusneo](https://statusneo.com/the-significance-of-parameter-efficient-fine-tuning-peft-with-lora-and-qlora/)

Goals:

- Reduce memory and compute.
- Keep most of the base model frozen.
- Allow many specialized variants without duplicating entire models.

Common PEFT families:

- Adapters, prefix-tuning, LoRA, etc. [statusneo](https://statusneo.com/the-significance-of-parameter-efficient-fine-tuning-peft-with-lora-and-qlora/)

### 5.3 LoRA (Low-Rank Adaptation)

Core idea:

- Instead of updating the full weight matrix \(W\), represent the update as a low-rank decomposition \(W + A B^\top\), where \(A\) and \(B\) are small trainable matrices. [databricks](https://www.databricks.com/blog/efficient-fine-tuning-lora-guide-llms)
- Original weights stay frozen; only the low-rank “delta” is learned.

Benefits: [databricks](https://www.databricks.com/blog/efficient-fine-tuning-lora-guide-llms)

- Only a small percentage of parameters are trainable (often < 1–5% or even lower).
- Dramatically reduces memory footprint and training cost while preserving most performance.
- Easy to combine different LoRA adapters for different tasks without duplicating the base model.

This is why LoRA is widely used in open-source LLM fine-tuning.

### 5.4 QLoRA

QLoRA extends LoRA with aggressive quantization: [geeksforgeeks](https://www.geeksforgeeks.org/deep-learning/fine-tuning-using-lora-and-qlora/)

- Quantize the base model to lower precision (e.g., 4-bit) for storage and forward passes.
- Attach LoRA adapters in higher precision (e.g., 16-bit) and train only those. [geeksforgeeks](https://www.geeksforgeeks.org/deep-learning/fine-tuning-using-lora-and-qlora/)
- Result: can fine-tune very large models on a single or few consumer-grade GPUs with minimal performance loss. [geeksforgeeks](https://www.geeksforgeeks.org/deep-learning/fine-tuning-using-lora-and-qlora/)

Interview phrasing:  
“QLoRA packs the base model into low-precision to save memory, then uses LoRA adapters in higher precision to learn task-specific behavior, giving near full-finetune performance at a fraction of cost.”

***

## 6. Strategy Selection (Fine-Tuning Choices)

### 6.1 Choosing the right technique

Think in terms of constraints:

- Constraint: compute budget (GPU memory, time).
- Constraint: number of tasks and variants.
- Goal: performance vs cost.

Rough decision table:

| Scenario                          | Recommended approach           |
|----------------------------------|--------------------------------|
| Maximum possible performance, big budget | Full fine-tuning             |
| Good performance, moderate budget        | LoRA (or other PEFT)        |
| Very tight GPU RAM, many tasks           | QLoRA or similar PEFT       |
| Need many task-specific variants         | PEFT (LoRA, adapters)       |

 [databricks](https://www.databricks.com/blog/efficient-fine-tuning-lora-guide-llms)

Interview line:  
“In a startup with limited GPUs, I’d default to PEFT (often LoRA/QLoRA). Full fine-tuning makes sense mainly when you operate at foundation-model scale with big infra.”

***

## 7. Research Areas Mentioned

### 7.1 NLP

Natural Language Processing covers understanding and generating text:

- Tasks: classification, NER, QA, summarization, translation, dialogue.
- LLMs have unified many NLP tasks via prompting and few-shot learning.

### 7.2 Computer Vision

Models that process images/videos:

- Tasks: classification, detection, segmentation, image generation.
- Increasingly integrated with text (e.g., text-to-image models, vision-language models).

### 7.3 Speech

Covers audio and spoken language:

- Speech recognition (ASR).
- Text-to-speech (TTS).
- Voice assistants and multimodal conversational systems.

### 7.4 Multimodal

Models that jointly process text + other modalities (image, audio, video):

- Examples: image captioning, visual QA, video understanding, chat over images.
- Typically share a common representation space across modalities.

### 7.5 Core ML / Deep Learning

Foundational topics:

- Architectures (transformers, CNNs, RNNs).
- Optimization (SGD variants, Adam).
- Regularization, generalization, representation learning.

Interview tip:  
Position your experience as “I’ve applied core deep learning ideas in one or more of these domains, and I understand how they now connect via multimodal foundation models.”

***

## 8. Cloud Computing Platforms

### 8.1 Why cloud matters for LLMs

Training and serving foundation models require:

- High-end GPUs/TPUs.
- Massive storage and bandwidth.
- Autoscaling, monitoring, and security.

Cloud platforms (AWS, Azure, GCP, etc.) provide:

- Managed GPU/TPU instances.
- Managed container/orchestration (EKS, AKS, GKE).
- Managed storage, networking, logging, and security services. [aws.amazon](https://aws.amazon.com/what-is/foundation-models/)

### 8.2 Typical LLM use of cloud

- Training:
  - Distributed training on GPU clusters.
  - Using spot instances or reserved capacity to control cost.
- Fine-tuning:
  - Shorter jobs with PEFT techniques on smaller clusters.
- Serving:
  - Inference endpoints (REST/GRPC) behind load balancers.
  - Autoscaling based on QPS and latency targets.
- MLOps:
  - Model registry, CI/CD, observability, access control, data pipelines.

Interview soundbite:  
“Modern LLM systems are cloud-native: training, fine-tuning, deployment, and monitoring are all integrated into cloud infrastructure for elasticity, cost control, and reliability.”

***
