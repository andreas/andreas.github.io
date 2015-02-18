---
layout: post
title: Envelope Encryption with Amazon KMS
redirect_from: /2015/01/04/envelope-encryption-with-amazon-kms/
---

[Amazon Key Management Service](http://aws.amazon.com/kms/) is a service for creating and controlling encryption keys in a safe manner, using [Hardware Security Modules](http://en.wikipedia.org/wiki/Hardware_security_module) under the hood. KMS also offers hassle-free yearly key rotation, and logs all key usage to [CloudTrail](http://aws.amazon.com/cloudtrail/) by default.

Using KMS, there are generally two ways to encrypt data: use the [Encrypt endpoint](http://docs.aws.amazon.com/kms/latest/APIReference/API_Encrypt.html) or use *envelope encryption*. This posts tries to provide an overview of the two.

The code snippet beneath will be the scaffold for our example code, which reads the sensitive data from stdin, encrypts it, prints the ciphertext and then decrypts it again:

{% highlight go linenos %}
import (
    "fmt"
    "io/ioutil"
    "os"
    "time"

    "github.com/awslabs/aws-sdk-go/aws"
    "github.com/awslabs/aws-sdk-go/gen/kms"
)

const (
    region = "eu-west-1"         // <-- change if needed
    keyId = "INSERT-KEY-ID-HERE" // <-- insert key id here
)

func main() {
    plaintext, err := ioutil.ReadAll(os.Stdin)
    if err != nil {
        panic(err)
    }

    // Create client
    creds, err := aws.ProfileCreds("", "", 10*time.Minute)
    if err != nil {
        panic(err)
    }
    kmsClient := kms.New(creds, region, nil)

    // Encrypt plaintext
    encrypted, err := Encrypt(kmsClient, plaintext) // <-- to be implemented
    if err != nil {
        panic(err)
    }
    fmt.Println("Encrypted: ", encrypted)

    // Decrypt ciphertext
    decrypted, err := Decrypt(kmsClient, encrypted) // <-- to be implemented
    if err != nil {
        panic(err)
    }
    fmt.Println("Decrypted: ", decrypted)
}
{% endhighlight go %}

The code assumes you have a [Amazon credentials file](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-config-files) in `~/.aws/credentials`, that you're in region `eu-west-1` (line 12), and that you have a KMS key id (line 13). If you don't have a KMS key, [here's how to create one](http://docs.aws.amazon.com/kms/latest/developerguide/create-keys.html).

## Encrypt Endpoint

Using the [Encrypt endpoint](http://docs.aws.amazon.com/kms/latest/APIReference/API_Encrypt.html) and [Decrypt endpoint](http://docs.aws.amazon.com/kms/latest/APIReference/API_Decrypt.html), it's easy to get started with protecting your data, and your application is never exposed to any keys or crypto. The caveats are latency, cost and an Amazon-imposed limit of 4KB on the size of the plaintext. This is enough for encrypting an RSA key, a database password or credit card information, though. Here's an implementation of `Encrypt` and `Decrypt` from the scaffold above, which uses these endpoints:

{% highlight go linenos %}
func Encrypt(kmsClient *kms.KMS, plaintext []byte) ([]byte, error)
    encryptRsp, err := kmsClient.Encrypt(&kms.EncryptRequest{
        Plaintext: plaintext,
        KeyID:     aws.String(keyId),
    })
    if err != nil {
        panic(err)
    }
    return encryptRsp.CiphertextBlob, nil
}

func Decrypt(kmsClient *kms.KMS, ciphertext []byte) ([]byte, error) {
    decryptRsp, err := kms.Decrypt(&kms.DecryptRequest{
        CiphertextBlob: ciphertext,
    })
    if err != nil {
        panic(err)
    }
    return decryptRsp.Plaintext, nil
}
{% endhighlight go %}

As you can tell it's quite simple.

## Envelope Encryption

KMS customer master keys are never exposed to the user, so they cannot be used for encrypting data locally. Instead, KMS offers the ability to generate new keys, which are exposed both in plaintext and in encrypted form. Data keys can then be used for encrypting locally, thus sidestepping the 4kB limit and the roundtrip to KMS. This is what is described as "envelope encryption" in the [KMS documentation](http://docs.aws.amazon.com/kms/latest/developerguide/workflow.html) . In more detail, the technique is as follows:

1. Generate a data key with the [GenerateDataKey endpoint](http://docs.aws.amazon.com/kms/latest/APIReference/API_GenerateDataKey.html) to obtain a key both as plaintext and ciphertext (encrypted).
2. Use the plaintext key to encrypt the payload.
3. Transmit the encrypted key along with the encrypted payload.
4. To decrypt the data, use KMS to decrypt the encrypted key and then decrypt the payload with the plaintext key.

To actually encrypt the message (step 2), the application will have to use a separate encryption scheme. The following example will be using [secretbox](http://godoc.org/golang.org/x/crypto/nacl/secretbox) from [NaCl](http://nacl.cr.yp.to/). To encrypt or decrypt a message with secretbox, you need both a key (secret) and a nonce (random data, not secret). The encrypted message will then be bundled up together with the key as encrypted with KMS and the nonce.

Here's the code to implement the above scheme:

{% highlight go linenos %}
import (
    "bytes"
    "encoding/gob"

    "golang.org/x/crypto/nacl/secretbox"
)

const (
    keyLength   = 32
    nonceLength = 24
)

type payload struct {
    Key     []byte
    Nonce   [nonceLength]byte
    Message []byte
}

func Encrypt(kmsClient *kms.KMS, plaintext []byte) ([]byte, error) {
    // Generate data key
    rsp, err := kmsClient.GenerateDataKey(&kms.GenerateDataKeyRequest{
        KeyID:         aws.String(keyId),
        NumberOfBytes: aws.Integer(keyLength),
    })
    if err != nil {
        return nil, err
    }

    // Initialize payload
    p := &payload{
        Key:   rsp.CiphertextBlob,
        Nonce: [nonceLength]byte{},
    }

    // Set nonce
    if _, err = rand.Read(&p.Nonce[:]); err != nil {
        return nil, err
    }

    // Create key
    key := &[keyLength]byte{}
    copy(key[:], rsp.Plaintext)

    // Encrypt message
    p.Message = secretbox.Seal(p.Message, plaintext, &p.Nonce, key)

		// Encode to bytes
    buf := &bytes.Buffer{}
    if err := gob.NewEncoder(buf).Encode(p); err != nil {
        return nil, err
    }

    return buf.Bytes(), nil
}

func Decrypt(kmsClient *kms.KMS, ciphertext []byte) ([]byte, error) {
    // Decode bytes to a payload
    var p payload
    gob.NewDecoder(bytes.NewReader(ciphertext)).Decode(&p)

    // Decrypt key
    decryptRsp, err := kmsClient.Decrypt(&kms.DecryptRequest{
        CiphertextBlob: p.Key,
    })
    if err != nil {
        return nil, err
    }

    // Create key
    key := &[keyLength]byte{}
    copy(key[:], decryptRsp.Plaintext)

    // Decrypt message
    var plaintext []byte
    plaintext, ok := secretbox.Open(plaintext, p.Message, &p.Nonce, key)
    if !ok {
        return nil, fmt.Errorf("Failed to open secretbox")
    }
    return plaintext, nil
}
{% endhighlight %}

Note that this code generates a new data key for each encrypted value, but the data key could just as well be generated once and reused. For simplicity, I've used [gob](http://golang.org/pkg/encoding/gob) to encode a payload of key, nonce and message to bytes.

If you want to give it a try, you can [download the sample code](https://gist.github.com/andreas/eea322d73ca21e66b55b). Run it like this after you've changed the key id:

```
> echo "My very secret data" | go run envelope_encryption.go
```

As evident from the two code samples above, envelope encryption is more complex than using the encrypt/decrypt endpoints, but depending on your use-case, it might be a requirement for speed or size of data.
