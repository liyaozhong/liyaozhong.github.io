---
layout: post
title: "PPO,GRPO,DPO"
subtitle: "PPO通过限制调整次数来确保结果不会偏离上次结果太多（稳定进步）
GRPO是PPO的扩展，没有价值函数，通过对比组内平均成绩来提高
DPO不打分，没有奖励模型，直接根据偏好数据调整"
tags: [LLM, Fine Tune, RL]
comments: false
---

### **Key Terms**

Here’s a summary of the key terms in the baking analogy:

- **Policy Model:** Contestant — bakes cakes (generates outputs).
    - **Model Parameters:** Baking strategies — determine the flavor and style of each cake.
    - **Output:** Cake — represents the model’s response or strategic behavior.
- **Reward Model:** Professional Judge ****— scores the cakes and provides feedback to guide improvements.
- **Value Function (Model):** The **Prophet or Potential Evaluator** — estimates the long-term expected returns from the current state.
- **Reference Model:** A classic recipe — the timeless formula each baker aims to follow or improve upon.
- **Binary Cross-Entropy Loss and KL Divergence Constraints:** Recipe’s characteristics — ensure the cake stays true to the core features of the original recipe.
- **Preference Data:** The audience — decides which cakes are the most popular, guiding bakers to refine their craft accordingly.

Figure 1: PPO, GRPO and DPO. [[Source1](https://arxiv.org/pdf/2402.03300v3)], [[Source2](https://arxiv.org/pdf/2305.18290v3)].

![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fe70a6de4-9f06-43ad-b13f-8833dafb71e6_2207x2265.png)

### **PPO: A Baking Competition with Judges**

Imagine a baking competition where each contestant (**the** **training model**) must bake a cake based on a classic recipe (**reference model**), and present it to a professional judge (**reward model**) for scoring. In addition, the potential evaluator (**value function**) assesses contestants' long-term potential by considering their skills, learning ability, growth prospects, and the expected value of their future work.

The judges rate the cakes based on appearance, aroma, and taste—the higher the score, the better the cake.

Contestants adjust their baking strategies (**model parameters**) based on the judges’ feedback, tweaking recipes, baking times, or temperatures to achieve higher scores in future rounds. This iterative process mirrors **reinforcement learning (RL).**

What makes PPO unique is that it limits how much contestants can change their strategies after each round. This prevents drastic shifts that could lead to worse results, ensuring steady progress.

Hiring expert judges is expensive, scoring takes time, and each contestant needs multiple rounds of feedback to improve. This makes training costly and inefficient.

Figure 2: PPO and GRPO. [[Source](https://arxiv.org/pdf/2402.03300v3)].

![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F1935e657-90fe-454c-a99e-0d3cf908a95d_2197x1022.png)

### **GRPO: A Baking Party with Peer Reviews**

Imagine a baking party where students (**from multiple samples of the training model**) are divided into different **groups**, and a professional judge (**reward model**) tastes and rates each one. Building on a classic recipe (**reference model**), the students improve their baking skills by comparing their results to the **group average** and learning from one another. There’s no prophet (**value function**) predicting each group’s future potential, as the focus is on mutual learning and progress within the group.

However, GRPO still relies on a professional judge (**reward model)** to score each output. If the reward model is biased, it can affect the overall effectiveness of the training. In addition, scoring within the group might lack consistency, especially if there’s a wide variation in the quality of outputs.

### **DPO: A Baking Show with Live Audience Voting**

DPO is like a baking show where each contestant (**policy model**) bakes two different cakes based on the same theme, following a classic recipe (**reference model**). Instead of scoring the cakes, the audience votes for their favorite (**preference data**).

The show’s producers then adjust each contestant’s recipe, increasing the proportion of ingredients from the preferred cake and reducing those from the less popular one. The extent of these adjustments depends on the voting margin and the recipe’s characteristics (**binary cross-entropy** **loss and KL divergence constraints**).

In the end, contestants create cakes that retain the classic flavor while appealing more to the audience’s taste.

Figure 3: DPO. [[Source](https://arxiv.org/pdf/2305.18290v3)].

![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F37023466-77a4-41c5-8b94-58e6bf2ce1e1_2198x1184.png)

Compared to PPO, DPO relies solely on audience feedback (**preference data**) and is more efficient by removing the need for reinforcement learning (**RL**), a professional judge (**explicit reward model**), and a potential evaluator (**explicit value function**), significantly reducing computational costs.

However, its performance heavily depends on the quality of the audience (**preference data**) and requires a classic recipe (**reference model**) during training to prevent strategy drift and maintain stable outputs.