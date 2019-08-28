---
layout: post
title: SwampCTF Writeup - Weak AES
---

It's been 6 months I haven't been playing CTFs, mostly due to me graduating INSA and having to leave its academic team InSecurity. But I'm back in the game, so let's celebrate by breaking some crypto !

This was the first crypto task of SwampCTF called "We Three Keys".

```python
#!/usr/bin/env python2
from Crypto.Cipher import AES
from keys import key1, key2, key3

def pkcs7_pad(msg):
    val = 16 - (len(msg) % 16)
    if val == 0:
        val = 16
    pad_data = msg + (chr(val) * val)
    return pad_data

def encrypt_message(key, IV):
    print("What message woud you like to encrypt (in hex)?")
    ptxt = raw_input("<= ")
    ptxt = pkcs7_pad(ptxt.decode('hex'))
    cipher = AES.new(key, AES.MODE_CBC, IV)
    ctxt = cipher.encrypt(ptxt)
    print ctxt.encode('hex')

def decrypt_message(key, IV):
    print("What message would you like to decrypt (in hex)?")
    ctxt = raw_input("<= ")
    ctxt = ctxt.decode('hex')
    if (len(ctxt) % 16) != 0:
        print "What a fake message, I see through your lies"
        return
    cipher = AES.new(key, AES.MODE_CBC, IV)
    ptxt = cipher.decrypt(ctxt)
    print ptxt.encode('hex')

def new_key():
    print("Which key would you like to use now? All 3 are great")
    key_opt = str(raw_input("<= "))
    if key_opt == "1":
        key = key1
    elif key_opt == "2":
        key = key2
    elif key_opt == "3":
        key = key3
    else:
        print("Still no, pick a real key plz")
        exit()
    return key

def main():
    print("Hello! We present you with the future kings, we three keys!")
    print("Pick your key, and pick wisely!")
    key_opt = str(raw_input("<= "))
    if key_opt == "1":
        key = key1
    elif key_opt == "2":
        key = key2
    elif key_opt == "3":
        key = key3
    else:
        print("Come on, I said we have 3!")
        exit()
    while True:
        print("1) Encrypt a message")
        print("2) Decrypt a message")
        print("3) Choose a new key")
        print("4) Exit")
        choice = str(raw_input("<= "))
        if choice == "1":
            encrypt_message(key, key)
        elif choice == "2":
            decrypt_message(key, key)
        elif choice == "3":
            key = new_key()
        else:
            exit()


if __name__=='__main__':
    main()
```

So far so good, even the first challenge looks like some actual cryptography and not some "hurr durr caesar base64 vigenere" kind of bullshit (I swear I'm not such a hater in real life).

If you are a crypto beginner and have trouble following along this post, don't worry. I have planned another article about cryptography basics, I have no idea when it will come out but it will at some point. Stay tuned !

I'm already digressing, great. The python code is pretty straightforward without any obfuscation, it uses the common package PyCrypto. We can already sense this challenge will be pure crypto, without much reverse engineering. I believe that the right mix of crypto and reverse makes for the best CTF challenges on earth, but is hard to achieve.

We can see the code uses AES, which is a famous cryptographic function. But the most important part here is the use of `AES.MODE_CBC`, which is where the vulnerability lies. I reckon I've probably lost half of the readers by talking about AES and CBC, so let's backtrack a little and explain what these two words mean.

## AES - Advanced Encryption Standard

This is probably the only encryption algorithm that is used more than RSA (both are extremely useful but do not serve the same purposes). AES has two main defining properties :

 - **Block cipher** : This algorithm takes a fixed size input (in this case, 16 bytes) called the plaintext, and spits an output of the same size called the ciphertext. If you're familiar with RSA, that one isn't a block cipher because it accepts ciphertexts of any size smaller than it modulus.
 - **Symmetric** : It means that the same key is used for both encryption and decryption.
 
Here is a visualization of how AES encrypts one block of data.

![aes_base]({{ site.baseurl }}/images/2019-04-swamp/aes_base.png)

Note that the output data looks like random noise, and that's the point of the encryption. If there were any information in the output that could lead to information about the input, the algorithm is broken. Luckily, AES isn't broken (yet), so the output bytes seem completely random.

The inverse operation takes a ciphertext and the key, and outputs the original plaintext. It's impossible (if used correctly) to guess the original plaintext without having the key.

![aes_decrypt]({{ site.baseurl }}/images/2019-04-swamp/aes_decrypt.png)

It's safe to assume in CTFs that we are not breaking the AES algorithm itself, but rather its implementation (choice of keys, compression, padding, ...). Actually, AES is only a building block of the challenge, but we should be focusing on entering through open doors instead of breaking through the concrete walls. Remember the `AES.MODE_CBC` ? It defines how the data is actually encrypted with the block cipher. Let's have a deeper look into it.

## Block cipher modes of operation

When operating with a block cipher, we need to cut the data in individual pieces and encrypt them because AES can only encrypt 16 bytes at a time.

Well, that's quite easy isn't it ? We can simply break the text into smaller chunks of 16 bytes and encrypt them one by one ! Congratulations, we just invented something called ECB.

#### ECB - Electronic Codebook

ECB is the retarded cousin of CBC, and here's why: let's imagine we are encrypting the message `Today's code: 4975384264852. Bye` using AES-ECB and sharing it with an ally :

![ecb1]({{ site.baseurl }}/images/2019-04-swamp/ecb1.png)

In this case the encryption is done correctly, and there is no way for an enemy intercepting the encrypted message to recover the ciphertext. However, fast forward a few weeks, we're sending a different code `Today's code: 4935412269921. Bye` encrypted with the same key.

![ecb2]({{ site.baseurl }}/images/2019-04-swamp/ecb2.png)

If you have a closer look at the ciphertext on the left, you will notice it's exactly the same as the previous one because the block's input is the same ! This might reveal some precious information on your code (in this case, the first 2 digits of the code) if someone intercepts the ciphertext and has enough information about older plaintexts. The supposedly perfect cryptosystem we invented has turned into a mediocre cryptosystem which can leak information. Using ECB is the easiest and fastest way to encrypt long plaintexts with block ciphers, but it's recommended to use another way of chaining blocks, such as CBC.

#### CBC - Cipher Block Chaining

I've talked about CBC enough that you must be dying to know what it's all about. It's time for me to reveal the ingredients to the CBC secret sauce that holds all the blocks together !

It's actually quite simple : a random value (called the initialization vector or IV) is mixed with the first plaintext before encryption. Then, we mix the first ciphertext with the second plaintext and encrypt the mix, then the second ciphertext with the third plaintext, and so on.

In practice, the mixing is done with XOR, which is a reversible transformation. If the IV is chosen randomly and never reused, the cascading property (aka butterfly effect in cryptography) of AES makes the cipher unbreakable.

![cbc1]({{ site.baseurl }}/images/2019-04-swamp/cbc1.png)

The IV is transmitted along with the ciphertext, making each transmission 16 bytes longer.

#### CBC decryption

The decryption algorithm is more complicated than the straightforward block-per-block ECB decryption, but we can understand how to reverse the operations made in the encryption. The mixing operation has to be reversible, because we need to untangle the ciphertext and the plaintext from the AES decryption's output.

XOR has several properties (for example, XOR is its own inverse, and 0 is its neutral element). This means that if we XOR the decrypted data (equal to `plaintext_block[i] xor ciphertext_block[i-1]`) with `ciphertext_block[i-1]`, we get

`(plaintext_block[i] xor ciphertext_block[i-1]) xor ciphertext_block[i-1]`

`= plaintext_block[i] xor (ciphertext_block[i-1] xor ciphertext_block[i-1])`

`= plaintext_block[i] xor 0`

`= plaintext_block[i]`

So we have successfully recovered the plaintext. The full operation is detailed here :

![cbc2]({{ site.baseurl }}/images/2019-04-swamp/cbc2.png)

Now, we have all the knowledge needed to find an attack on the cryptosystem.

I won't get into PKCS7 padding which is also implemented in the challenge. The only thing you need to know about PKCS7 is that it's used to make the ciphertext length a multiple of 16 so the data can nicely be split into blocks without the last one being too short.

## Applied attack on bad AES-CBC

Now if you remember what I said, the IV has to be chosen perfectly at random and never reused. But have a look at the implementation :

```python
def encrypt_message(key, IV):
    print("What message woud you like to encrypt (in hex)?")
    ptxt = raw_input("<= ")
    ptxt = pkcs7_pad(ptxt.decode('hex'))
    cipher = AES.new(key, AES.MODE_CBC, IV)
    ctxt = cipher.encrypt(ptxt)
    print ctxt.encode('hex')
    
# (later in the source)
encrypt_message(key, key)
```

Not only is the IV reused for all messages, it also contains a value that should be kept secret ! To attack this, we don't even need to use the encrypt function - let's look at what happens if we decrypt a made-up ciphertext full of null bytes :

![attack]({{ site.baseurl }}/images/2019-04-swamp/attack.png)

Since the only thing that determines the output of AES encryption/decryption is the data and the key, all three AES decryption blocks output the same data.

However, outputs 1, 2 and 3 are different because they are not XORed with the same values, but the data which exits each block is exactly the same.

Output 1 corresponds to `decryptAES(0000000000000000, key) xor IV`, whereas outputs 2 and 3 are `decryptAES(0000000000000000, key) xor 0000000000000000`. Let's verify in practice that output 2 and 3 are equal (I added spaces between blocks for clarity).

```
$ nc chal1.swampctf.com 1441

Hello! We present you with the future kings, we three keys!
Pick your key, and pick wisely!
<= 1
1) Encrypt a message
2) Decrypt a message
3) Choose a new key
4) Exit
<= 2
What message would you like to decrypt (in hex)?
<= 00000000000000000000000000000000 00000000000000000000000000000000 00000000000000000000000000000000
9c53fc4f03020389898c77d7cdaa0f74 fa3f9d28787533fed6fb1fe3b9f56340 fa3f9d28787533fed6fb1fe3b9f56340
```

The value of `decryptAES(0000000000000000, key)` is not predictable, even with such a special input. However, we can use properties of XOR to recover the IV. If we XOR the first two output blocks, the result will be

`Output1 xor Output2 = (decryptAES(0000000000000000, key) xor IV) xor (decryptAES(0000000000000000, key) xor 0000000000000000)`

XORing with zeros does nothing (0 is the identity element of XOR), so we have

`Output1 xor Output2 = (decryptAES(0000000000000000, key) xor IV) xor decryptAES(0000000000000000, key)`

`Output1 xor Output2 = IV xor decryptAES(0000000000000000, key) xor decryptAES(0000000000000000, key)`

`Output1 xor Output2 = IV xor (decryptAES(0000000000000000, key) xor decryptAES(0000000000000000, key))`

`Output1 xor Output2 = IV xor 0000000000000000`

`Output1 xor Output2 = IV`

By XORing the first two output blocks, we can successfully recover the Initialization Vector which contains a secret value. The only thing left to do is convert it back to a printable string :

```python
o1 = '9c53fc4f03020389898c77d7cdaa0f74'
o2 = 'fa3f9d28787533fed6fb1fe3b9f56340'

flag = ''
for i in range(0,32,2):
    flag += chr(int(o1[i:i+2],16)^int(o2[i:i+2],16))
print flag
```

Rinse and repeat for the two other keys, and we get the whole flag : `flag{w0w_wh4t_l4zy_k3yz_much_w34k_crypt0_f41ls!}`

I didn't explain all the tiny crypto details in this post as I wanted to keep it short and beginner-friendly, but it's a very interesting topic with lots of concepts accessible to anyone. In a future blog post, I will show more cryptographic principles in detail, starting from zero.