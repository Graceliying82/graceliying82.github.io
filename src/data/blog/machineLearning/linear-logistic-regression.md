---
title: "The Foundations of Learning: A Deep Dive into Linear and Logistic Regression"
author: Grace Li
pubDatetime: 2026-03-14T10:45:00Z
postSlug: linear-logistic-regression-fundamentals
featured: false
draft: false
tags:
  - machine-learning
  - ai
  - fundamentals
  - math
  - tutorial
description: "Before diving into neural networks, you have to master the basics. Here’s a breakdown of the two most important building blocks in machine learning: Linear and Logistic Regression."
---

Welcome to the first post in my new series on Machine Learning and Neural Networks! Before we start building complex brains with billions of parameters, we need to understand the fundamental ways a machine actually "learns" from data. 

In this post, we're going back to basics with **Linear Regression** and **Logistic Regression**. While they sound like cousins, they are used for very different tasks.

## 1. The Basic Concept

### Linear Regression: Predicting the Continuous
Imagine you're trying to predict the price of a house based on its square footage. As the size increases, the price generally goes up in a straight-ish line. This is a **regression** problem because we are predicting a **continuous value** (price).

The math is simple:
$$y = wx + b$$

*   **$y$**: The prediction (e.g., house price).
*   **$x$**: The input feature (e.g., square footage).
*   **$w$**: The weight (how much each square foot adds to the price).
*   **$b$**: The bias (the "starting price" even for 0 sq ft).

The goal of Linear Regression is to find the best $w$ and $b$ that minimizes the distance between our line and the actual data points.

### Logistic Regression: The Gatekeeper of Classification
Now, imagine instead of predicting the *price*, you want to predict if a tumor is **Malignant** or **Benign**. This is a **classification** problem. We aren't looking for a continuous number; we're looking for a probability (Yes/No, 0 or 1).

We still use the same linear formula ($wx + b$), but we wrap it in a special function called the **Sigmoid Function** ($\sigma$):

$$\sigma(z) = \frac{1}{1 + e^{-z}}$$

This function squashes any input value to be between **0 and 1**, which we can interpret as a probability. If the result is > 0.5, we predict "Class 1"; otherwise, "Class 0".

## 2. Loss vs. Cost Function: Measuring Failure

In ML, we don't just guess; we measure how wrong we are and try to improve. This is where the **Loss** and **Cost** functions come in.

> [!NOTE]
> **Loss** refers to the error for a **single** training example.  
> **Cost** is the average of the loss over the **entire** dataset.

### Linear Regression: Mean Squared Error (MSE)
For linear regression, we use MSE. We take the difference between the prediction ($\hat{y}$) and the actual value ($y$), square it (to keep it positive), and average it.

$$J(w,b) = \frac{1}{2m} \sum_{i=1}^{m} (\hat{y}^{(i)} - y^{(i)})^2$$

### Logistic Regression: Binary Cross-Entropy (Log Loss)
We can't use MSE for Logistic Regression because the Sigmoid function would make the cost curve "wavy" (non-convex), meaning we might get stuck in a bad spot. Instead, we use **Log Loss**:

$$L(\hat{y}, y) = -(y \log(\hat{y}) + (1-y) \log(1-\hat{y}))$$

*   If the true label is $y=1$ and we predict $\hat{y} \approx 1$, the loss is near zero.
*   If we predict $\hat{y} \approx 0$ when $y=1$, the $-\log(\hat{y})$ term goes to infinity. We are heavily penalized for being "confidently wrong."

## 3. How to Train a Model: The Gradient Descent

Training a model is just an optimization problem: "Find the values of $w$ and $b$ that make the Cost Function as small as possible."

The most common way to do this is **Gradient Descent**.

1.  **Initialize**: Start with random values for $w$ and $b$.
2.  **Calculate the Gradient**: Find the derivative of the cost function relative to $w$ and $b$. This tells you the "slope" – which direction is "downhill"?
3.  **Update**: Take a small step in the downhill direction.
4.  **Repeat**: Keep stepping until the cost stops decreasing.

The size of that step is controlled by the **Learning Rate** ($\alpha$).
*   If $\alpha$ is too small, training takes forever.
*   If $\alpha$ is too large, you might overshoot the minimum and never converge.

---

## What's Next?
Mastering these two algorithms is the key to understanding everything else in AI. Neural networks are essentially just layers and layers of these mathematical functions stacked on top of each other.

In the next post, we'll talk about **Vectorization** – the secret sauce that makes these calculations run fast on your computer.

---
*Happy Learning!*
