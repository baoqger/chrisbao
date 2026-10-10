---
title: "Running ONNX GenAI Model on GPU - How to Set Up Driver, CUDA 13, cuDNN, and NuGet"
date: 2026-10-10 13:05:00
tags: ONNX, Edge AI, GPU, CUDA, dotnet
keywords: ONNX Runtime GenAI, CUDA 13, cuDNN, Phi-3, Microsoft.ML.OnnxRuntimeGenAI.Cuda
---


## Background

In my previous two articles, I walked through edge AI development on the .NET stack using the ONNX model format. Those posts relied on **CPU** hardware. In this article, I will extend that path and use a **GPU** for GenAI inference. If you want to know how much faster inference can get on GPU—or how to set up a local **CUDA** environment on Windows—this post is for you. Let’s get started.

<img src="/images/running-onnx-genai-model-on-gpu-how-to-set-up-driver-cuda-13-cudnn-and-nuget/Accelerating_AI_with_GPU_Inference.png" alt="" >

## Model

As I mentioned in the first article in this series, the **Phi-3-Mini-4K-Instruct-onnx** model ships in several variants for different compute targets: **`cuda`**, **`directml`**, and **`cpu_and_mobile`**.

<img src="/images/running-onnx-genai-model-on-gpu-how-to-set-up-driver-cuda-13-cudnn-and-nuget/Pasted%20image%2020260911153901.png" alt="" >

In that earlier post we used the **`cpu_and_mobile`** build. Today we switch to the **`cuda`** folder. We still use a **quantized** model; the weight file is about **2.3 GB**.

<img src="/images/running-onnx-genai-model-on-gpu-how-to-set-up-driver-cuda-13-cudnn-and-nuget/Pasted%20image%2020260928214416.png" alt="" >

The model’s `genai_config.json` makes the execution provider explicit—`provider_options` is set to **CUDA**:

```json
{
    "model": {
        "bos_token_id": 1,
        "context_length": 4096,
        "decoder": {
            "session_options": {
                "log_id": "onnxruntime-genai",
                "provider_options": [
                    {
                        "cuda": {
                            "enable_cuda_graph": "0"
                        }
                    }
                ]
            },
            "filename": "phi3-mini-4k-instruct-cuda-int4-rtn-block-32.onnx",
            "head_size": 96,
            "hidden_size": 3072,
            "inputs": {
                "input_ids": "input_ids",
                "attention_mask": "attention_mask",
                "past_key_names": "past_key_values.%d.key",
                "past_value_names": "past_key_values.%d.value"
            },
            "outputs": {
                "logits": "logits",
                "present_key_names": "present.%d.key",
                "present_value_names": "present.%d.value"
            },
            "num_attention_heads": 32,
            "num_hidden_layers": 32,
            "num_key_value_heads": 32
        },
        "eos_token_id": [
            32000,
            32001,
            32007
        ],
        "pad_token_id": 32000,
        "type": "phi3",
        "vocab_size": 32064
    },
    "search": {
        "diversity_penalty": 0.0,
        "do_sample": false,
        "early_stopping": true,
        "length_penalty": 1.0,
        "max_length": 4096,
        "min_length": 0,
        "no_repeat_ngram_size": 0,
        "num_beams": 1,
        "num_return_sequences": 1,
        "past_present_share_buffer": true,
        "repetition_penalty": 1.0,
        "temperature": 1.0,
        "top_k": 1,
        "top_p": 1.0
    }
}
```

Next step, let’s look at the hardware I used for this demo.

## GPU hardware

My machine has an **NVIDIA RTX PRO 4000 Blackwell Generation Laptop GPU**:

<img src="/images/running-onnx-genai-model-on-gpu-how-to-set-up-driver-cuda-13-cudnn-and-nuget/Pasted%20image%2020260928220251.png" alt="" >

It has **16 GB** of VRAM—enough headroom for small models like **Phi-3-Mini-4K-Instruct-onnx**. Great, right?

## Environment

To run this ONNX model on GPU with CUDA, you need **four layers** of software on top of the hardware:

- **NVIDIA driver** — required for any GPU workload; nothing special to add here.
- **CUDA Toolkit** — the CUDA SDK (compiler, runtime, debugger, and related tools). **Pick a toolkit version that matches your GPU generation.** On my Blackwell laptop I use **CUDA 13.4**. Installing the toolkit takes time, so confirm your GPU architecture and the ORT/GenAI package requirements **before** you install.
- **cuDNN** — NVIDIA’s deep learning primitives library; ONNX Runtime GPU depends on it.
- **`Microsoft.ML.OnnxRuntimeGenAI.Cuda`** — the .NET package for CUDA. It pulls in native dependencies such as **ONNX Runtime GPU**. Package version must align with your CUDA stack—for example, **0.16.0** targets the **CUDA 13** line.

Setting this up involves trial and error; give yourself some patience. Once everything is in place, we can move to the demo.

## Demo

Switching from CPU to GPU does **not** require rewriting your agent. You mainly change how the model is loaded.

**Lower-level API:**

```csharp
using Microsoft.ML.OnnxRuntimeGenAI;

string modelPath = @"C:\path\to\cuda-model";
using Config config = new Config(modelPath);

config.ClearProviders();
config.AppendProvider("cuda");

using Model model = new Model(config);
```

**Higher-level API** (`Microsoft.Extensions.AI`):

```csharp
using Microsoft.Extensions.AI;
using Microsoft.ML.OnnxRuntimeGenAI;

string modelPath = @"C:\path\to\cuda-model";

using Config config = new Config(modelPath);
config.ClearProviders();
config.AppendProvider("cuda");

using var client = new OnnxRuntimeGenAIChatClient(config);

ChatResponse response = await client.GetResponseAsync("Hello", new ChatOptions
{
    MaxOutputTokens = 64,
});
```

The important line is **`AppendProvider("cuda")`**.

As we mentioned above, the model folder must be the **`cuda`** variant (not `cpu_and_mobile`). The config snippet simply tells ONNX Runtime GenAI to use the CUDA execution provider at runtime.

### CPU vs GPU throughput

First, the **CPU** run:

<img src="/images/running-onnx-genai-model-on-gpu-how-to-set-up-driver-cuda-13-cudnn-and-nuget/Pasted%20image%2020260921132134.png" alt="" >

**438 tokens** in **25 seconds** — about **17 tokens/s**.

Then the **GPU** run:

<img src="/images/running-onnx-genai-model-on-gpu-how-to-set-up-driver-cuda-13-cudnn-and-nuget/Pasted%20image%2020260921232228.png" alt="" >

Same **438 tokens** in **3.5 seconds** — about **121 tokens/s**. That is roughly a **7–8×** speedup. Smart design, right? Most of the win comes from moving matmul-heavy work off the CPU and onto the GPU execution provider.

## Summary

After reading this post, you should understand:

- Why the Hugging Face **`cuda`** ONNX build pairs with **`OnnxRuntimeGenAI.Cuda`** on .NET.
- The **four-layer** stack: driver → CUDA Toolkit → cuDNN → GenAI CUDA NuGet—and why versions must match (especially on newer GPUs such as Blackwell).
- How little code changes when you switch providers: **`ClearProviders()`** + **`AppendProvider("cuda")`**.
- What kind of speedup to expect on a small quantized Phi-3 model when CPU and GPU are compared fairly.
