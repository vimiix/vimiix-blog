---
title: 'Golang实现Google在RTB广告中价格加密方案'
date: 2019-11-13 16:22:48
categories: 'golang'
tags: ['solution', 'golang', 'rtb']
---

## RTB

RTB 广告是一种实时竞价广告，就是在针对每个广告位有展示机会的时候，会实时多方竞价，价格最有优势的广告主会竞得这次展示机会，在媒体测在拿到素材的时候，需将本次成交的价格，上报给指定的监控服务器，这时就需要将实时价格按照指定的加密方案加密后，替换 GET 链接中的请求参数中的价格宏来上报。

官方给出的源代码有 Java 和 C++ 版本, 下载地址： <https://code.google.com/archive/p/privatedatacommunicationprotocol/source/default/source>

本文主要通过 golang 来实现的 google 的价格[加密方案](https://developers.google.com/authorized-buyers/rtb/response-guide/decrypt-price#encryption-scheme)。

<!--more-->

### Source Code

```go
package main

/*
golang 实现 google 的rtb 价格加密方案
https://developers.google.com/authorized-buyers/rtb/response-guide/decrypt-price#encryption-scheme
*/

import (
 "crypto/hmac"
 "crypto/md5"
 "crypto/sha1"
 "encoding/base64"
 "encoding/binary"
 "encoding/hex"
 "fmt"
 "hash"
 "math"
 "strings"
)

const (
 PayloadSize    = 8
 InitVectorSize = 16
 SignatureSize  = 4
 EKey           = ""
 IKey           = ""
 UTF8           = "utf-8"
)

func AddBase64Padding(base64Input string) string {
 var b64 string
 b64 = base64Input
 if i := len(b64) % 4; i != 0 {
  b64 += strings.Repeat("=", 4-i)
 }
 return b64
}

func CreateHmac(key, mode string, isBase64 bool) (hash.Hash, error) {
 var b64DecodedKey []byte
 var k []byte
 var err error
 if isBase64 {
  b64DecodedKey, err = base64.URLEncoding.DecodeString(AddBase64Padding(key))
  if err == nil {
   key = string(b64DecodedKey[:])
  }
 }
 if mode == UTF8 {
  k = []byte(key)
 } else {
  k, err = hex.DecodeString(key)
 }
 if err != nil {
  return nil, err
 }
 return hmac.New(sha1.New, k), nil
}

func HmacSum(_hmac hash.Hash, buf []byte) []byte {
 _hmac.Reset()
 _hmac.Write(buf)
 return _hmac.Sum(nil)
}

func Hmac(key string, buf []byte) ([]byte, error) {
 _hmac, err := CreateHmac(key, UTF8, true)
 if err != nil {
  err = fmt.Errorf("jzt/encrypt: create hmac error, %s", err.Error())
  return nil, err
 }
 return HmacSum(_hmac, buf), nil
}

func EncryptPrice(price float64) (string, error) {
 var (
  iv         = make([]byte, InitVectorSize)
  encoded    = make([]byte, PayloadSize)
  signature  = make([]byte, SignatureSize)
  priceBytes = make([]byte, PayloadSize)
 )

 sum := md5.Sum([]byte(""))
 copy(iv[:], sum[:])

 // pad = hmac(e_key, iv)  // first 8 bytes
 pad, err := Hmac(EKey, iv[:])
 if err != nil {
  return "", err
 }
 pad = pad[:PayloadSize]

 bits := math.Float64bits(price)
 binary.BigEndian.PutUint64(priceBytes, bits)
 // enc_price = pad <xor> price
 for i := range priceBytes {
  encoded[i] = pad[i] ^ priceBytes[i]
 }

 // signature = hmac(i_key, data || iv), first 4 bytes
 sig, err := Hmac(IKey, append(priceBytes[:], iv[:]...))
 if err != nil {
  return "", err
 }
 signature = sig[:SignatureSize]

 // final_message = WebSafeBase64Encode( iv || enc_price || signature )
 finalMessage := strings.TrimRight(
  base64.URLEncoding.EncodeToString(append(append(iv[:], encoded[:]...), signature[:]...)),
  "=")
 return finalMessage, nil
}

func DecryptPrice(finalMessage string) (float64, error) {
 var err error
 var errPrice float64
 encryptedPrice := AddBase64Padding(finalMessage)
 decoded, err := base64.URLEncoding.DecodeString(encryptedPrice)
 if err != nil {
  return errPrice, err
 }
 var (
  iv         = make([]byte, InitVectorSize)
  p          = make([]byte, PayloadSize)
  signature  = make([]byte, SignatureSize)
  priceBytes = make([]byte, PayloadSize)
 )

 copy(iv[:], decoded[0:16])
 copy(p[:], decoded[16:24])
 copy(signature[:], decoded[24:28])

 pad, err := Hmac(EKey, iv[:])
 if err != nil {
  return errPrice, err
 }
 pad = pad[:PayloadSize]
 for i := range p {
  priceBytes[i] = pad[i] ^ p[i]
 }

 sig, err := Hmac(IKey, append(priceBytes[:], iv[:]...))
 if err != nil {
  return errPrice, err
 }
 sig = sig[:SignatureSize]
 for i := range sig {
  if sig[i] != signature[i] {
   return errPrice, fmt.Errorf("jzt/decrypt: Failed to decrypt, got:%s ,expect:%s", string(sig), string(signature))
  }
 }

 price := math.Float64frombits(binary.BigEndian.Uint64(priceBytes))
 return price, nil
}


func main() {
 var result string
 var err error
 var price float64
 result, err = EncryptPrice(0.11)
 if err != nil {
  err = fmt.Errorf("Encryption failed. Error : %s", err)
 }
 fmt.Println(result)

 price, err = DecryptPrice("1B2M2Y8AsgTpgAmY7PhCfumh5TxN_8-DUn4uHg")
 if err != nil {
  err = fmt.Errorf("Encryption failed. Error : %s", err)
 }
 fmt.Println(price)
}
```
