---
layout: post
title: AES ECB Pattern Vunerability
date: 2017-12-31 04:00:00.000
tags: code cryptography
author: Cameron A. Craig
cover: assets/images/2017-12-31-aes-image-patterns/bruges.jpg
---

An investgation into a pitfall of the AES ECB cipher and how this is overcome in AES CBC.

### Title Image

The title image of this post is a photo of Bruges, the capital city of Belgium.
Rijndael (now known as AES) was developed by Vincent *Rij*men and Joan *Dae*men, both from Belgium.
This image is licensed under Creative Commons Attribution-Share Alike 3.0 Unported, and originated from
[Elke Wetzig](https://commons.wikimedia.org/wiki/User:Elya) on
[Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Bruegge_huidenvettersplein.jpg).

## Introduction

This article discusses the AES ECB pattern vunerability by worked examples using OpenSSL and AF_ALG.
CBC and ECB modes are different in the way blocks feed into the next block.
AES ECB is the simpler of the two AES variants under comparison, this runs the risk of people using ECB mode inapproprately.
The inappropriate use of ECB runs the risk of making the ciphertext vunerable to attack.

The pattern vunerability is illustarted by incrypting PPM images, allowing the plaintext and ciphertext images to be visualed side-by-side.

All code used in this article is licensed under GNU GPL v2 unless stated otherwise.
Please get in touch if you find this article useful.

## The AES specification

[The American Encryption Standard (AES)](https://csrc.nist.gov/csrc/media/publications/fips/197/final/documents/fips-197.pdf)
describes an algorithm that takes in a key and a plaintext as inputs, and a ciphertext as the output.
This algorithm takes a fixed-size plaintext of 16 bytes, and outputs ciphertext of the same fixed size.
The key length can be any of the following values: 128 bits, 192 bits, 256 bits (it's common for key sizes to be specifies in bits rather than bytes).
This same algorithm is used for each mode of encryption.
The difference in each mode is the way each block is connected to the next.

## How the blocks connect

The following diagrams illustrate the two AES modes used:

### ECB (Electronic Code Book)

![ECB Block Diagram](assets/images/2017-12-31-aes-image-patterns/ecb_block_diagram.png)

### CBC (Cipher Block Chaining)

![CBC Block Diagram](assets/images/2017-12-31-aes-image-patterns/cbc_block_diagram.png)

## Image Format

For this article we use the PPM image format, in the binary form.
This format works well for these examples, as the image data section is valid no matter how messed up the data is.
The PPM format is also extremely simple.
We don't encrpyt the header as this contains crucial image size data that we need to keep the same.
When we use the PPM format in ASCII form, the encrypted image becomes invalid as the ciphertext may not be a valid ASCII value.

## ECB Pattern Vunerability

The reason an image can still be recognised after ECB encryption is because 16 bytes of the same plaintext will always become the same ciphertext,
no matter the position with respect to the rest of the blocks.
So a group of white pixels next to some blue pixels will looks the same in the ciphertext domain as a group of white pixels next to some pink pixels.

The CBC algorithm defeats this vunerability by XORing the ciphertext of the previous blobk with the key, creating dependencies between neighbouring blocks. Now white pixels will be encrypted to a different colour depending on the pixel before that, and before that.

## Encrypting PPM images using OpenSSL

The following bash script can be used to encrypt and decrypt an image.

```

```

The PPM pixel size is 3 bytes (one byte for each colour, RGB), and the AES block size is 16 bytes. That is
why areas of the same colour in the plaintext domain appear to have a striped
pattern in the ciphertext domain.

This diagram shows how each pixel of a PPM image is encrypted, creating a pattern in the encrypted image:

| Pixel # |1 | 1 | 1 | 2| 2 | 2 |3 | 3 | 3 | 4 | 4 | 4 | 5 | 5 | 5 | 6 |
|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|
| Colour | r | g | b | r | g | b | r | g | b | r | g | b | r | g | b | r |
| Byte Count | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |

Each cell represents a byte. Five full pixels can fit into a block, with the last pixel overlapping into the next block.

### Resulting images

<amp-img width="600" height="300" layout="responsive" src="assets/images/2017-12-31-aes-image-patterns/secret_message.png"></amp-img>
<amp-img width="600" height="300" layout="responsive" src="assets/images/2017-12-31-aes-image-patterns/secret_message.ppm.ecb.png"></amp-img>
<amp-img width="600" height="300" layout="responsive" src="assets/images/2017-12-31-aes-image-patterns/secret_message.ppm.cbc.png"></amp-img>

<amp-img width="128" height="162" layout="responsive" src="assets/images/2017-12-31-aes-image-patterns/Tux.png"></amp-img>
<amp-img width="128" height="162" layout="responsive" src="assets/images/2017-12-31-aes-image-patterns/Tux.ppm.ecb.png"></amp-img>



<figure class="ampstart-image-with-caption m0 relative mb4">
	<amp-img width="128" height="162" layout="responsive" src="assets/images/2017-12-31-aes-image-patterns/Tux.ppm.cbc.png"></amp-img>
	<figcaption class="h5 mt1 px3">Tux.png encrypted using AES CBC</figcaption>
</figure>



## Encypting PPM Images using AF_ALG

The follwing C code acesses the kernel crypto API using the AF_ALG userspace API.

```
```

In order to remove the patterning effect of misaligned pixels relative to the
AES blocks, we can encrypt each pixel in a seperate block, and pad the remainder
of the block with zeroes. This ciphering technique is visualised in the table below:

| Pixel # |1 | 1 | 1 | | | | | | | | | | | | | |
|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|
| Colour/0 | r | g | b | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| Byte Count | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |

By encrypting each pixel in a seperate AES block, we remove the paterning effect.
This results in each pixel colour mapping to the same pixel colour in the ciphertext domain.

To acheive our one-pixel-per-block encryption we use null padding. This fills the remainder of the 16-byte block with zeros.

### Resulting images