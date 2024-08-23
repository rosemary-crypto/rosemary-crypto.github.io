---
layout: post
title: Introduction to Secret Sharing
date: 2024-08-20 11:12:00-0400
description: An introduction to Shamir's Secret Sharing and its applications
tags: cryptography, secret sharing, TSS
categories: cryptography
featured: true
---

## Secret Sharing

I wouldn’t call myself a secretive person, but I do value my privacy. If you’re like me, you’ve probably found yourself holding onto a secret so tightly that you start feeling like it might just burst out. But instead of spilling the whole thing to one person, you decide to share bits and pieces with different people. This way, no single person knows the full story, and you feel a bit more secure.

Secret sharing works in a similar way. Introduced by Adi Shamir and George Blakley independently in 1979 (yes, it’s been around for a while, but like a good classic, it’s still relevant), secret sharing is all about distributing a secret among a group of participants. The twist? No single participant has any meaningful information on their own. However, when a sufficient number of these participants come together, they can reconstruct the original secret.

In this post, we'll explore Shamir's version of secret sharing, a powerful cryptographic tool that splits a secret into pieces, or "shares." These shares are individually useless, but when you have enough of them (we’ll call this number the threshold), you can piece together the original secret.

## Mathematical Definition of Shamir's Secret Sharing

Let’s get a bit more technical. SSS is sometimes referred to as the $$(k, n)$$ threshold secret sharing scheme. Here, $$n$$ is the number of shares into which the secret is divided, and $$k$$ is the minimum number of shares required to reconstruct the secret. Essentially, $$k$$ participants must come together to unlock the secret. If you have fewer than $$k$$ shares, you're out of luck; the secret remains hidden.

But how does this actually work? The secret is hidden inside a mathematical function called a polynomial. A polynomial is just a mathematical expression involving a sum of powers of $$x$$ multiplied by coefficients. In Shamir’s scheme, we use a polynomial of degree $$k-1$$, which ensures that any group of $$k$$ or more participants can reconstruct the secret.

$$
f(x) = s_0 + s_1x + \dots + s_{k-1}x^{k-1} \mod p
$$

Lagrange’s interpolation can be used to reconstruct the secret. The Lagrange interpolation formula for reconstructing the polynomial (and thus the secret) from these shares is:

$$
q(x) = \sum_{i=1}^{k} y_i \cdot L_i(x)
$$

where $$y_i$$ are the share values, and $$L_i(x)$$ are the Lagrange basis polynomials, defined as:

$$
L_i(x) = \prod_{\substack{1 \le j \le k \\ j \ne i}} \frac{x - x_j}{x_i - x_j}
$$

## Why Modular Arithmetic and Prime Numbers?

- **Modular arithmetic** is a system of arithmetic for integers, where numbers wrap around after they reach a certain value, $$p$$ (the modulus). Think of it like a clock: after 12 comes 1 again. In cryptography, modular arithmetic helps keep numbers within a specific range, making it more manageable and secure.
  
- **Prime numbers** are crucial because they ensure certain properties that are vital for cryptographic security:
  - **Unique Inverses**: In modular arithmetic with a prime modulus $$p$$, every non-zero number has a unique multiplicative inverse. This means that for any non-zero number $$a$$ within the range $$[1, p-1]$$, there exists a number $$b$$ such that $$(a \cdot b) \mod p = 1$$. This property is crucial because it allows division operations to be performed in modular arithmetic.
  - **Avoiding Redundancies**: Prime moduli help avoid redundancies and ensure that every possible result of an operation is unique within the range $$[0, p-1]$$. This uniqueness is critical in cryptography because it prevents different numbers from being treated as equivalent, which could otherwise lead to vulnerabilities.
  - **Mathematical Stability**: Using a prime number ensures that the structure of the arithmetic remains stable, making it easier to predict and manage the behavior of calculations, which is why primes are often used in cryptographic protocols.

## Basic Example with Numbers

Let’s start simple. Imagine you have a secret number, and you want to split it among five people so that any three of them can come together and recover it. You’ll create a polynomial, and each person gets a point on this polynomial as their share. Only when three or more people pool their points together can they figure out what the original number (the secret) was.

### Generating the Shares

Let's take a $$(2, 5)$$ threshold scheme to divide the secret 298 and let the prime number be 307. (I chose 307 since it is a prime number larger than 298)

1. **Set up the polynomial**
   - Degree: $$k-1 = 1$$
   - Form: $$q(x) = s + a_1 \cdot x \mod p = 298 + a_1 \cdot x \mod 307$$

2. **Choose a random coefficient** $$a_1$$ in the range $$[0, p)$$. Let's choose $$a_1 = 123$$.

3. **Calculate the shares**: Now, we can calculate the shares at 5 non-zero points $$x = 1, 2, 3, 4, 5$$ as:
   - Share 1: $$ (1, 114) $$
   - Share 2: $$ (2, 237) $$
   - Share 3: $$ (3, 53) $$
   - Share 4: $$ (4, 176) $$
   - Share 5: $$ (5, 299) $$

You now have 5 shares: $$ (1, 114), (2, 237), (3, 53), (4, 176), (5, 299) $$.

### Reconstructing the Secret Using Lagrange Interpolation

Let’s now reconstruct the secret using Lagrange interpolation. Since this is a $$(2, 5)$$ threshold scheme, we need only two of the shares to reconstruct the secret. We’ll use Share 1 $$(1, 114)$$ and Share 2 $$(2, 237)$$.

1. **Calculate the Lagrange basis**
   - For our example, with $$x_1 = 1$$ and $$x_2 = 2$$, the Lagrange basis polynomials at $$x = 0$$ (which gives us the secret) are:

   $$
   L_1(0) = \frac{0 - 2}{1 - 2} = \frac{-2}{-1} = 2
   $$

   $$
   L_2(0) = \frac{0 - 1}{2 - 1} = \frac{-1}{1} = -1
   $$

2. **Calculate the Secret**
   - Now, we use these Lagrange basis polynomials to reconstruct the secret:

   $$
   q(0) = y_1 \cdot L_1(0) + y_2 \cdot L_2(0)
   $$

   - Substituting the values from the shares:

   $$
   q(0) = 114 \cdot 2 + 237 \cdot (-1) = 228 - 237 = -9
   $$

   - Since we're working in modular arithmetic with $$p = 307$$, we take the result modulo 307:

   $$
   q(0) = -9 \mod 307 = 298
   $$

Thus, the reconstructed secret is $$s = 298$$, which matches our original secret.

## Shamir's Secret Sharing Using a Secret Image

To really bring this to life, let's take it a step further. Instead of just splitting a number, let’s split an image. Lets take an example of (3,5) threshold secret sharing as shown in the figure.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="/assets/img/secret_sharing/3_5_SecretImageSharing.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    ((3,5) Threshold Secret Image Sharing)
</div>

### How It Works

- **Image Splitting**: Each pixel’s grayscale value in the image is treated as a secret. Shamir’s Secret Sharing algorithm is applied to split each pixel’s value into multiple shares. For instance, in a $$(3,5)$$ threshold scheme, the image is divided into 5 shares, but at least 3 shares are needed to reconstruct the image.

- **Share Generation**: For each pixel, shares are generated based on a random polynomial where the pixel value is the constant term. These shares are then distributed, with each share forming part of a different image, each appearing as random noise.

- **Image Reconstruction**: To reconstruct the image, a minimum of 3 shares are combined using Lagrange interpolation to recover the original pixel values and, thus, the entire image.

### Code Implementation

The complete implementation of this concept is provided in the repository. The code is available in both C++ and Rust:

- **C++ Version**:
  - Located in the `cpp` directory, this version uses OpenCV for image processing.

- **Rust Version**:
  - Located in the `rust` directory.

The code is accessible on [GitHub](https://github.com/rosemary-crypto/Shamir-sSecretSharing) for further exploration and application.

## Significance in Modern Applications

Now, why should you care about this old-school cryptography trick? Well, let’s dive into the world of Threshold Signature Schemes (TSS). Imagine you’re working on a distributed system, like a blockchain. In such systems, it's crucial that a group of participants can collaboratively sign off on transactions or documents. But here’s the catch: you don’t want any single participant holding the entire signing key, because that’s a single point of failure and a potential security nightmare.

TSS comes to the rescue by allowing a group of participants to generate a digital signature on a message without anyone holding the full key. Instead, the key is distributed, and only a subset of participants—what we call the threshold number—needs to come together to produce a valid signature. This method not only enhances security by eliminating single points of failure but also builds trust, as no single party controls the entire signing process.

However, TSS more complex than SSS, but that’s why we’re here. Before diving into TSS, it’s important to understand SSS and its limitations so we can appreciate the need for more advanced techniques like TSS.
