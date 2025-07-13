ECDSA Secp256k1 Sign-Verify サンプルコード
===

## cryptography サンプルコード

```bash
pip install cryptography eth-utils eth-hash[pycryptodome]
```

```python
from cryptography.exceptions import InvalidSignature
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import ec
from cryptography.hazmat.primitives.asymmetric.utils import Prehashed
from eth_utils import keccak

# ---------------------------
# キーペア生成（secp256k1）
# ---------------------------
private_key = ec.generate_private_key(ec.SECP256K1())
public_key = private_key.public_key()

# ---------------------------
# メッセージに署名
# ---------------------------
message = b"Hello, Ethereum world!"

# 署名（ECDSA with SHA256）
signature = private_key.sign(message, ec.ECDSA(hashes.SHA256()))
print("署名（hex）:", signature.hex())

# ---------------------------
# 検証
# ---------------------------
try:
    public_key.verify(signature, message, ec.ECDSA(hashes.SHA256()))
    print("✅ 署名は有効です。")
except InvalidSignature:
    print("❌ 署名が無効です。")


# ----------------------------
# メッセージのハッシュ値に署名
# ----------------------------
digest = keccak(message)  # Keccak-256 (Ethereumの仕様)

# -----------------------
# Prehashed 署名
# -----------------------
# ハッシュサイズが256ビットなので Prehashed(hashes.SHA256())
signature = private_key.sign(digest, ec.ECDSA(Prehashed(hashes.SHA256())))
print("署名（hex）:", signature.hex())

# -----------------------
# 検証
# -----------------------
try:
    public_key.verify(signature, digest, ec.ECDSA(Prehashed(hashes.SHA256())))
    print("✅ 検証成功（ハッシュ値に対して）")
except InvalidSignature:
    print("❌ 検証失敗")
```