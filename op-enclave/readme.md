# 使用 Docker 登录
echo $GITHUB_TOKEN | docker login ghcr.io -u sunchaoqun --password-stdin

# 为现有镜像添加 GHCR 标签
docker tag op-enclave:latest ghcr.io/sunchaoqun/op-enclave:latest

# 推送镜像
docker push ghcr.io/sunchaoqun/op-enclave:latest

go run ./cmd/server/main.go

nitro-cli terminate-enclave --all

nitro-cli run-enclave --eif-path op-enclave-copy.eif --enclave-cid 16 --memory 1024 --cpu-count 2 --debug-mode --attach-console

# 获取解密公钥并保存
DECRYPTION_KEY=$(curl -s -X POST -H "Content-Type: application/json" http://localhost:7333 \
  --data '{"id":1,"jsonrpc":"2.0","method":"enclave_decryptionPublicKey","params":[]}' | jq -r '.result' | sed 's/^0x//')

# 检查是否成功获取到公钥
echo $DECRYPTION_KEY


# 使用解密公钥加密您的签名者私钥
# 这一步需要使用 OpenSSL 或其他加密工具
# 将解密公钥保存为 PEM 文件
echo "$DECRYPTION_KEY" | xxd -r -p > decryption_key.der
openssl rsa -pubin -inform DER -in decryption_key.der -out decryption_key.pem


PRIVATE_KEY=$(python3 -c "from web3 import Web3; w = Web3().eth.account.create(); print(w.key.hex()[2:])")
echo $PRIVATE_KEY

PRIVATE_KEY=$(openssl rand -hex 32)
echo $PRIVATE_KEY

# 将私钥转换为二进制并加密
echo -n "$PRIVATE_KEY" | xxd -r -p > private_key.bin
ENCRYPTED_KEY=$(openssl pkeyutl -encrypt -pubin -inkey decryption_key.pem -in private_key.bin | xxd -p -c 1000)

# 检查加密后的密钥
echo $ENCRYPTED_KEY

# 设置签名者密钥
curl -X POST -H "Content-Type: application/json" http://localhost:7333 \
  --data "{\"id\":2,\"jsonrpc\":\"2.0\",\"method\":\"enclave_setSignerKey\",\"params\":[\"0x$ENCRYPTED_KEY\"]}"


#验证公钥
SIGNER_PUBLIC_KEY=$(curl -s -X POST -H "Content-Type: application/json" http://localhost:7333 \
  --data '{"id":3,"jsonrpc":"2.0","method":"enclave_signerPublicKey","params":[]}' | jq -r '.result')

echo $SIGNER_PUBLIC_KEY


python3 get_public_key.py $PRIVATE_KEY


python3 -c "from web3 import Web3; from eth_account import Account; Account.enable_unaudited_hdwallet_features(); private_key='0x$PRIVATE_KEY'; account = Account.from_key(private_key); public_key = Account._keys.PrivateKey(Web3.to_bytes(hexstr=private_key)).public_key; print('0x' + public_key.to_hex()[2:])"

curl -d '{"id":0,"jsonrpc":"2.0","method":"enclave_signerAttestation"}' -H "Content-Type: application/json" http://localhost:7333

