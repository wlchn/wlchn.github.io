---
title: "RSA-SHA256-非对称签名与验证-sign-verify----JavaScript,-Ruby,-Golang"
date: 2018-08-06T00:00:00+08:00
---

## RSA 主要用法

1. 公钥加密(encrypt)，私钥解密(decrypt)
2. 私钥签名(sign)，公钥验证(verify)

网上讲述 RSA 原理的文字很多，很少涉及签名验证的实现。
第一种比较常见，本文主要是第二种方法的实现(JavaScript, Ruby, Golang)。
本文的 signature 最终结果都为 hex。

## JavaScript

sign

```
import { KJUR } from 'jsrsasign'

const msg = "hello world"
const privateKey = "-----BEGIN RSA PRIVATE KEY-----(your private key)"

const sig = new KJUR.crypto.Signature({ "alg": "SHA256withRSA" });
sig.init(privateKey);
sig.updateString(msg)
const signatureHex = sig.sign()
```

verify

```
const publicKey = "-----BEGIN PUBLIC KEY-----(your public key)"

const sig2 = new KJUR.crypto.Signature({"alg": "SHA256withRSA"});
sig2.init(publicKey);
sig2.updateString(msg)
const isValid = sig2.verify(signatureHex)
```

## Ruby

sign

```
private_key = "-----BEGIN RSA PRIVATE KEY-----(your private key)"
msg = "hello world"

digest = OpenSSL::Digest::SHA256.new
pkey = OpenSSL::PKey::RSA.new(private_key)
signature = pkey.sign(digest, msg)
hex_signature = signature.unpack('H*').first
```

verify

```
public_key = "-----BEGIN PUBLIC KEY-----(your public key)"
msg = "hello world"

digest = OpenSSL::Digest::SHA256.new
pkey = OpenSSL::PKey::RSA.new(public_key)
pub_key = pkey.public_key
puts pub_key.verify(digest, signature, msg)
```

## Golang

sign

```
import (
	"io"
	"crypto/rand"
	"encoding/hex"
	"crypto/sha256"
	"crypto/rsa"
	"encoding/pem"
	)

msg := "hello world"

rng := rand.Reader
message := []byte(msg)
hashed := sha256.Sum256(message)
signature, err := rsa.SignPKCS1v15(rng, privateKey, crypto.SHA256, hashed[:])
signatureHex := hex.EncodeToString(signature)
```

verify

```
msg := "hello world"

sig, err := hex.DecodeString(signatureHex)
if err != nil {
  fmt.Printf( "err hex signature: %s\n", err)
  return false
}

hashed := sha256.Sum256([]byte(msg))

err = rsa.VerifyPKCS1v15(publicKey, crypto.SHA256, hashed[:], sig)
if err != nil {
  fmt.Fprintf(os.Stderr, "verify failed: %s\n", err)
  return false
}
return true
```
