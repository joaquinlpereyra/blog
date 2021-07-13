---
title: 'Solving Cryptopals with my own AES implementation'
date: 2021-07-12T21:22:18-03:00
draft: false
tags:
  - "cryptopgrahy"
  - "aes"
  - "cryptopals"
description: "A blog post describing how I solved the first two sets of Cryptopals using my own AES implementation"
---

A couple of years ago I started with the [Cryptopals challenges](https://cryptopals.com/). For those that don't know them, they are a collection of excercises on cryptography that take a very hands-on and yet provide very little guidance on how to solve them. These kind of practice has become my favorite: I really enjoy being presented a problem and having the freedom to solve it as best as I can.

Anyway. The challenges start quite easy, with a couple of excercises with XOR cyphers, which are extremely basic cyphers where `encryption(plaintext, key) = xor(plaintext, key) = decryption(plaintext, key)`. See how easy it is to bruteforce one using some basic frequency analysis.

```rust
pub fn xor_file() {
    let file = match File::open("./6.txt") {
        Ok(file) => file,
        Err(_) => panic!("file not found!"),
    };
    let buffer = BufReader::new(file);
    for encrypted_line in buffer.lines().map(|l| l.unwrap()) {
        let encrypted_bytes = hex::from_string(&encrypted_line).unwrap();
        // Bruteforce the key
        for i in 0..127 {
            let xored = bytes::repeating_xor(&encrypted_bytes, &[i as u8]);
            let text = match bytes::to_string(&xored) {
                Some(text) => text,
                None => continue, // most probably our secret is utf8, at least?
            };
            let score = frequency::analysis(&text);
            if score > 0.7 {
                println!("score: {}, text: {}, key: {}", score, text, i);
            }
        }
    }
}

```

That was the **hard** problem on set zero. Everything before it was very easy, requiring basic programming knowledge (you are OK with for-loops) and maybe knowing a couple of properties of XOR.

Then the next challenge appears simple: you are given a key, you are told it is for the AES cypher under ECB mode. You are given a file and you are told it's encoded in base64.

Simple enough unless you are me and you were set on using zero dependencies. That means implementing **base64** and **AES**. **base64** is somewhat easy and unremarkable, but **AES** was quite the journey. It took me a fair amount of work to get it working. I actually spent ~2 years on this, but they were off and on. Mostly off. But I can say that implementing **AES** was more frustrating than solving the challenges themselves.

Anyway, after all the work, I had a working **AES** implementation:

```rust
pub fn decrypt_yellow_submarine() {
    // https://cryptopals.com/sets/1/challenges/7

    let cipher_text = read_to_string("./7.txt").expect("could not read file");
    let cipher_text = cipher_text.replace('\n', "");
    let cipher_text = base64::decode(&cipher_text).unwrap();
    let cipher_key = "YELLOW SUBMARINE".as_bytes();

    let plain = symm::decrypt(&cipher_key, &cipher_text, symm::Mode::ECB, Padding::None);
    let plain_ascii = str::from_utf8(&plain).unwrap();
    print!("{}", plain_ascii);
}
```

Let's look at my implementation and see the interesting bits. 

So, first things first, you should know the difference between the algorithm and the mode.

**The algorithm** is AES, the [Advanced Encryption Standard](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard). It is a subset of `Rijndael`. `Rijndael` was presented when the NIST was hey `hey guys we need an advanced encryption standard` then the guys behind `Rijndael` were like `we have this` and the NIST was like `coolio so this is now AES`. The should explain the name. All right.

Then **the mode** is something entirely separate from the algorithm itself and should work for any block cypher... oh, I should now mention the difference between **block cyphers** and **stream cyphers**.

**Stream cyphers** encrypt an arbitrary amount of data and can encrypt one byte at a time. They need to generate a **keystream**, that is, an arbitrarily long key with which to encrypt the data.

**Block cyphers** take a fixed chunk of data, say, 16 bytes, or 8 bytes, or 92, whatever. But they are **fixed**. If my block cypher accepts 16 bytes of data and I want to encrypt 14 or 18, I'm screwed. The **key size** in block cyphers is also fixed.

Obviously **block cyphers** seem simpler, but also useless because why would one want to encrypt data of an exact size. That seems like a very niche use case. Well, you have two methods come to the rescue:

* **Padding**, which should be familiar for most developers. If I have `0xCOFFEEBABE` (5 bytes) as my data and my block cipher takes 6 bytes input, I can pad it somehow. **PKCS7** is a popular padding for cryptography which is self-contained: you don't need to store how many bytes you padded or where the pad beggins: that information is the pad itself. So `PKCS7(0xCOFFEEBABE, target=6) = 0xC0FFEEBABE01`, because I needed one byte. The downside is that if my data already is six bytes, it will double its size: `PKCS7(0xCOFFEEBABE01, target=6) = 0xCOFFEEBABE01060606060606`.

* **Modes of operation**, which solve the inverse problem: I want to pad _more_ than my algorithm is capalbe of. **Modes of operation** is a very bad name for a simple idea: how can we concatenate each block produced by calling my algorithm with the acceptable amount of bytes? So how do I transform the separate entities `AES(first_chunk), AES(second_chunk), AES(third_chunk)` into a single message and how do I do it securely?

Note that both **padding** and a **mode** are needed in conjunction in most cases. The `first_chunk` in `AES(first_chunk)` will probably have to be padded.

So that's it about **modes**. So let's see how to implement modes. Lucily, the text is not padded (that means the plaintext had to have a length which is a multiple of 16, the AES block size). No padding means I can show my `PKCS7` implementation in another post. It is not that interesting anyway.

But there is a mode, and the mode is **ECB**. **ECB** or ****Electronic Code Book** (who comes up with these names?) is the simplest mode. It just... well, it basically does nothing. It just concatenates blocks togethers. 

```rust
/// An ECB mode of operation for an arbitrary block cypher
/// ECB mode will just encrypt every block and concatenate
/// the outputs to form the cyphertext. Idem for decryption.
/// Operation on the ECB mode require a working block cipher
/// and do not assume the plain or cipher text to be padded.
pub struct ECB<'a> {
    cipher: &'a mut dyn Cipher,
    block_size: usize,
}

impl<'a> ECB<'a> {
    pub fn new(cipher: &'a mut dyn Cipher) -> ECB<'a> {
        let block_size = cipher.get_block_size();
        ECB { cipher, block_size }
    }

    pub fn encrypt(&mut self, msg: &[u8]) -> Vec<u8> {
        let mut cipher_text = Vec::with_capacity(msg.len());

        for i in (0..msg.len()).step_by(self.block_size) {
            self.cipher.set_state(&msg[i..i + self.block_size]);
            cipher_text.append(&mut self.cipher.encrypt());
        }

        cipher_text
    }

    pub fn decrypt(&mut self, cipher_text: &[u8]) -> Vec<u8> {
        let mut plain_text = Vec::with_capacity(cipher_text.len());

        for i in (0..cipher_text.len()).step_by(self.block_size) {
            self.cipher.set_state(&cipher_text[i..i + self.block_size]);
            plain_text.append(&mut self.cipher.decrypt())
        }

        plain_text
    }
}
```

My implementation lies a little bit and would make you believe that `ECB` encrypts stuff. It does not. The encryption is done by a `cipher`. The name `encrypt` is there because that's what ends up happening under the hood.

Anyway, as you can see, the code is as simple as it can be: iterate over every chunk of `plain_text[i:i+block_size]`, then set `i = i+block_size`. Concat the results.

We only now need to understand **AES** encryption/decryption. Now, I have a confession to make here. Implementing **AES** was probably the most boring and futile excercise I've done. I became quite obsessed with finishing it so I did it. But I would not recomend it. The low-level details are uninteresting, and you won't become better at cryptography because you implemented it. A high-level overview is great and it is awesome to understand how it works, but the implementation details are just boring.

That's why I will just leave exactly that: a high level overview. I **won't** be showing how to implement the math so operations work inside the field of `FD(2^8)`, nor will I show exactly all the permutations that are happening under the hood, nor the `key expansion` that **AES** performs before encrypting your plaintext. If there's any kind of interest on it contact me at @agaricus_sec on Twitter and I can work on a post or maybe lending a hand, but if you are reading this you are probably as qualified as me to solve the problem at hand.

Well, then let's see how **AES** works. 

```rust
/// A low-level AES Cipher.
/// It provides the basic primitives of the AES algorithm.
/// Clients are supposed to check for sanity of input parameters
/// or the Cipher will either malfunction or just panic.
/// Sanity requirements:
/// - Key is either 16, 24 or 32 bytes long
/// - A call to set_state is made before trying to encrypt or decrypt
pub struct Cipher {
    nr: u8,
    state: Block,
    key: Key,
}

impl Cipher {
    // ...
    pub fn encrypt(&mut self) -> [u8; 16] {
        self.add_round_key(0);

        for round in 1..self.nr {
            self.substitute_bytes();
            self.shift_rows();
            self.mix_columns();
            self.add_round_key(round);
        }

        self.substitute_bytes();
        self.shift_rows();
        self.add_round_key(self.nr);

        self.state.flatten_into_u8()
    }

    pub fn decrypt(&mut self) -> [u8; 16] {
        self.add_round_key(self.nr);

        for round in 1..self.nr {
            self.inverse_shift_rows();
            self.inverse_substitute_bytes();
            self.add_round_key(self.nr - round);
            self.inverse_mix_columns();
        }

        self.inverse_shift_rows();
        self.inverse_substitute_bytes();
        self.add_round_key(0);

        self.state.flatten_into_u8()
    }

    pub fn set_state(&mut self, state: &[u8]) {
        let block = Block::new_from_u8([
            [state[0], state[1], state[2], state[3]],
            [state[4], state[5], state[6], state[7]],
            [state[8], state[9], state[10], state[11]],
            [state[12], state[13], state[14], state[15]],
        ]);
        self.state = block;
    }
}
```

Well, you can see that **AES** then has a couple of primite operations:

* `add_round_key`, which multiplies the key by the `round number` and a constant `Nb = 4`
* `substitute_bytes`, which uses a `SUBSTITUTION_BOX` or `SBOX` for short to scrammble the input. Every byte of the input is mapped into the `SBOX` is literally an array of arrays (a box) and replaced by the values on it.
* `shift_rows` will shift the rows in a block (an horizontal array of bytes inside a block, AES blocks have 4 rows and 4 columns). What this means is it will take the first byte of the first row and replace it by the second byte on the second row.
* `mix_columns` which does something similar for columns.

Then the inverse of those functions is also implemented and used in the decryptions.

And that's **AES**. It is so simple (basically shifting stuff around and replacing bytes by publicly known values) that its security should come as a surprise. I know it did for me. I'm still not intuitively-convinced it is not reversible without the key, although practice has shown it is, at least when using a sane mode (will come back to that later in another post).

I think this goes without saying, but of course my **AES** implementation is a toy. It is not safe, do not use it, dragons will eat you, etc.

I still have a lot to write about on this topic. I didn't even show yet how to break `ECB` mode, which you will be glad to discover can be completely, absolutely broken: you can recover the plaintext with only a access to an oracle that can encrypt an arbitrary plaintext. 

If you want to read more of my code implementing AES, go straight for [the repository](https://github.com/joaquinlpereyra/crypto).
