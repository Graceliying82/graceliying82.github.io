---
title: "Beyond the Line: Building Your First Shallow Neural Network"
author: Grace Li
pubDatetime: 2026-03-15T11:32:00Z
postSlug: shallow-neural-networks-scratch
featured: false
draft: false
tags:
  - machine-learning
  - neural-networks
  - fundamentals
  - python
  - math
description: "Why settle for a straight line? Learn how to stack hidden layers to build a shallow neural network that can learn complex patterns."
---

In our previous posts, we mastered **Linear** and **Logistic Regression**. We saw how these models find a "best-fit" line or boundary to separate data. But what happens when the data isn't linearly separable? What if the boundary needs to be a circle, a wave, or something even more complex?

Welcome to the world of **Neural Networks**.

## 1. What is a "Shallow" Neural Network?

A **Shallow Neural Network** is simply a network with a **single hidden layer** between the input and the output. 

While Logistic Regression is essentially a network with 0 hidden layers (it goes straight from input to output), adding just one hidden layer allows the model to learn **non-linear** features.

## 2. Key Concepts: The Building Blocks

Before we look at the math, we need to define the "levers" we can pull to make the network learn.

### Parameters: The Learned Weights
**Parameters** are the values that the network learns during training.
*   **Weights ($W$):** These represent the strength of the connection between neurons. Think of them as individual knobs the network turns to emphasize different features.
*   **Biases ($b$):** These allow the network to shift the activation function up or down, providing extra flexibility to fit the data.

### Hyperparameters: The Designer's Choice
**Hyperparameters** are values that *you*, the engineer, must set before training begins. They aren't learned; they are used to control the learning process.
*   **Learning Rate ($\alpha$):** How big of a step the network takes during optimization.
*   **Number of Hidden Units ($n_h$):** How "broad" your hidden layer is (more on this below).
*   **Iterations:** How many times the network looks at the training data.

## 3. Width vs. Depth: "Broader" vs. "Deeper"

When people talk about the "size" of a network, they usually refer to two dimensions:
*   **Width (Broader)**: This refers to the number of neurons (hidden units) in a single layer. A "broad" network has many neurons per layer, allowing it to remember more specific patterns in the training data.
*   **Depth (Deeper)**: This refers to the total number of layers. Adding more layers is what transforms a "Shallow" network into a "**Deep Neural Network**" (DNN).

Shallow networks are the perfect learning tool because they introduce the core math of deep learning without the complexity of managing dozens of layers.

## 4. The Architecture

A typical shallow network looks like this:

1.  **Input Layer ($x$):** Your raw features (e.g., pixel values, house size).
2.  **Hidden Layer ($a^{[1]}$):** A set of neurons that perform an intermediate calculation.
3.  **Output Layer ($a^{[2]}$):** The final prediction (e.g., probability of an image being a cat).

### The Mathematical Flow

For each layer, we perform two steps: a linear computation ($Z$) and an activation function ($A$).

**Layer 1:**
$$Z^{[1]} = W^{[1]}X + b^{[1]}$$
$$A^{[1]} = \text{ReLU}(Z^{[1]})$$

**Layer 2 (Output):**
$$Z^{[2]} = W^{[2]}A^{[1]} + b^{[2]}$$
$$A^{[2]} = \sigma(Z^{[2]})$$

### Choosing Your Activation Function

You might be wondering: "Which one should I use?" 

*   **ReLU (Rectified Linear Unit):** This is the **modern standard** for hidden layers. It's incredibly fast to compute and helps prevent the "vanishing gradient" problem (where the network stops learning because updates become too small). If you're unsure, **always start with ReLU**.
*   **Tanh:** Historically popular because it's zero-centered, but it can suffer from slow learning in very deep networks.
*   **Sigmoid ($\sigma$):** Rarely used in hidden layers today, but still the go-to for the final **output layer** in binary classification because it returns a clean probability between 0 and 1.

## 5. Vectorization: Keeping it Fast

As we discussed in the [last post](file:///Users/graceli/src/AntigravityProj/graceliying82.github.io/src/data/blog/machineLearning/vectorization.md), we never use `for` loops to iterate over training examples. Instead, we stack all $m$ examples into a matrix $X$.

### Single Example vs. Vectorized

To really understand why vectorization is so powerful, let's look at the transition from "one at a time" to "all at once."

| Step | Single Example $(i)$ | Vectorized (All $m$ examples) |
| :--- | :--- | :--- |
| **Input** | $x^{(i)}$ (vector) | $X = [x^{(1)}, x^{(2)}, \dots, x^{(m)}]$ (matrix) |
| **Linear Step** | $z^{(i)} = Wx^{(i)} + b$ | $Z = WX + b$ |
| **Activation** | $a^{(i)} = \sigma(z^{(i)})$ | $A = \sigma(Z)$ |
| **Shape of $A$** | $(n_{neurons}, 1)$ | $(n_{neurons}, m)$ |

By using the **Vectorized Version**, we calculate the results for all $m$ training examples in one single matrix operation. NumPy handles the heavy lifting, distributing the work across your CPU's cores or your GPU's thousands of parallel units.

## 6. Why it Works: Non-Linearity

If we didn't use an activation function (if $A = Z$), then stacking 100 layers would just be the same as a single linear regression! 

The **activation function** is the magic ingredient. By adding "kinks" and "curves" to the math, we allow the network to approximate almost any function. This is known as the **Universal Approximation Theorem**.

## 7. Implement It! (The Forward Pass)

In modern NumPy, we use the **`@` operator** (or `np.matmul`) for matrix multiplication. While `np.dot` still works for 2D matrices, `@` is more explicit and handles higher-dimensional arrays more consistently.

Here is a snippet showing the forward propagation using the `@` operator:

```python
import numpy as np

def relu(Z):
    return np.maximum(0, Z)

def forward_propagation(X, parameters):
    # Retrieve parameters
    W1 = parameters['W1']
    b1 = parameters['b1']
    W2 = parameters['W2']
    b2 = parameters['b2']
    
    # Layer 1: Linear (@) -> ReLU
    Z1 = W1 @ X + b1
    A1 = relu(Z1)
    
    # Layer 2: Linear (@) -> Sigmoid
    Z2 = W2 @ A1 + b2
    A2 = 1 / (1 + np.exp(-Z2)) # Sigmoid
    
    cache = {"Z1": Z1, "A1": A1, "Z2": Z2, "A2": A2}
    return A2, cache
```

