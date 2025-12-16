

## Asymmetric encryption (RSA in this case) simplified what I know so far without deep math knowledge:
### Checkpoint for later memory recovery regarding RSA 😅️

**NOTE: strict can and can nots are different from USE CASES, I list these below that**

**NOTE: These charachteristics are unique to RSA, and can differ based on the formula/cipher, but some concepts of course are general**

Pub keys can:
Encrypt a message<br>
Decrypt a message encrypted by private key
<br>
Pub keys can't:<br>
Decrypt a message encrypted by pub key itself (this would've compromised the security in it's core)
<br>
Private keys can:<br>
Encrypt a message<br>
Decrypt a message encrypted by pub key
<br>
Private keys can't:<br>
(this is not so important, focus on other things)<br>
Decrypt a message encrypted by private key itself. <br>

`Chart for what was desribed above`

## RSA Can / Can't Chart

| Operation | Possible? | Notes |
|-----------|-----------|-------|
| Public key encrypt → Private key decrypt | ✅ | Standard confidentiality |
| Private key encrypt → Public key decrypt | ✅ | Signing / verification |
| Public key decrypt public-encrypted | ❌ | Would break security |
| Private key decrypt private-encrypted | ❌ | Not meaningful, expected failure |


Look - in alternative world, private keys and public keys could switch places.

Since public key can decrypt encrypted messages by private key,

and private key can decrypt messages encrypted by public key (and other integrities also remain same).


## Use cases.
Since we named the private key private:

1. Encrypted communication

Here we use private key only for reading messages and handle public key to others.
People encrypt and send messages, we in our end can read these messages by decrypting them with our according private key.

So in here public key is the only thing exposed and public key can't decrypt what's encrypted with public key itself => so integrity remains.

2. Signing and verifying ownership (apps,software etc..)

In this case, we want people do be able to verify we are the producer of someapp.exe and it hasn't been tampered.
Here we can for example: get the hash of someapp.exe file, and sign it with our private key.

Then we list/expose the public key, and people having it, can check on the app if the file was tampered or no, by decrypting signed stuff and comparing with hash (also called signature verification). so we can host the signed app on different places, but the public key the user gets to verify the app signature should belong to legit vendor - us in this case.


## Test RSA keys private/public interchangability

```
# Create working directory and enter it
echo "Creating test directory..."
mkdir test_RSA_keys_interchangability && cd test_RSA_keys_interchangability

# Generate 4096-bit RSA private key
echo "Generating 4096-bit RSA private key..."
openssl genpkey -algorithm RSA -out priv.pem -pkeyopt rsa_keygen_bits:4096

# Extract public key from private key
echo "Extracting public key from private key..."
openssl rsa -in priv.pem -pubout -out pub.pem

# Create a sample message
echo "Creating sample message... (Hello world)"
echo "Hello world" > msg.txt

# Encrypt (sign) message with private key
echo "Signing message with private key... (encrypting the message with private key)"
openssl rsautl -sign -in msg.txt -inkey priv.pem -out msg.signed

# Decrypt (verify) message with public key
echo "Verifying signed message with public key... (decrypting the message with pub key)"
openssl rsautl -verify -in msg.signed -inkey pub.pem -pubin -out decrypted.txt

# Show decrypted content
echo "Decrypted content:"
cat decrypted.txt

```
