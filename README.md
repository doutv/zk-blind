# zk-blind

post anonymous confessions about your work place / organization in zero-knowledge!

`yarn` to install all dependencies.

## generate inputs

to generate inputs into `____.json`, replace `signature`, `msg`, and `ethAddress` in `node-ts scripts/generate_input.ts`. currently, this file will only generate inputs for OpenAI JWTs and JWTs generated by our dummy JWT generator application(https://get-jwt.vercel.app/), but feel free to add more public keys to support JWTs from different sites.

on the last line of `scripts/gen_inputs.ts`, edit the json file name.
```
ts-node scripts/generate_input.ts
```

## circuits
```sh
# Prepare the environment
source ~/.rapidsnark

# export LD_LIBRARY_PATH=/home/okxdex/data/zkdex-pap/services/rapidsnark/depends/pistache/build/src
# alias proverServer=/home/okxdex/data/zkdex-pap/services/rapidsnark/build_nodejs/proverServer
# alias prover=/home/okxdex/data/zkdex-pap/workspace/cliff/rapidsnark/package/bin/prover
# export REQ=/home/okxdex/data/zkdex-pap/services/rapidsnark/tools/request.js
# export PTAU=/home/okxdex/data/zkdex-pap/workspace/cliff/zkdex-plonky2-circom-poc/circom/e2e_tests/powersOfTau28_hez_final_24.ptau
```

These circuits check for (1) valid rsa signature, (2) that the message is a JWT, (3) ownership of a specific email domain, and (4) JWT expiration.

compile circuits in root project directory.
```
./shell_scripts/1_compile.sh
```

generate witness
```
./shell_scripts/2_gen_wtns.sh
```
make sure to edit the input json file name to the correct input file you generated in the generate inputs step.

phase 2 and getting full zkey + vkey
```
snarkjs groth16 setup ./build/jwt/jwt.r1cs $PTAU ./build/jwt/jwt_single.zkey

snarkjs zkey contribute ./build/jwt/jwt_single.zkey ./build/jwt/jwt_single1.zkey --name="1st Contributor Name" -v

snarkjs zkey export verificationkey ./build/jwt/jwt_single1.zkey ./build/jwt/verification_key.json
```

### Generate Proof
snarkjs
```sh
snarkjs groth16 prove ./build/jwt/jwt_single1.zkey ./build/jwt/witness.wtns ./build/jwt/proof.json ./build/jwt/public.json;
```

rapidsnark standalone mode
```sh
prover ./build/jwt/jwt_single1.zkey ./build/jwt/witness.wtns ./build/jwt/proof.json ./build/jwt/public.json;
```

rapidsnark server mode
```sh
# build cpp
cd build/jwt/jwt_cpp
make
cd ../..
cp ./build/jwt/jwt ./build/jwt_single1

# Copy witness
cp ./build/jwt/witness.wtns ./build/jwt_single1.wtns

# Start a new terminal and run the prover server
proverServer 9080 ./build/jwt/jwt_single1.zkey

# Request the prover server
# params: <input.json> <circuit_name>
node $REQ ./build/input_jwt_single1.json jwt_single1;
```

verify proof offchain
```
snarkjs groth16 verify ./build/jwt/verification_key.json ./build/jwt/public.json ./build/jwt/proof.json
```

generate verifier.sol
```
snarkjs zkey export solidityverifier ./build/jwt/jwt_single1.zkey contracts/Verifier.sol
```

run local hardhat test
```
npx hardhat test ./test/blind.test.js
```

deploy blind and verifier contracts
```
npx hardhat run ./scripts/deploy.js --network goerli
```

### Benchmark Different Prover
```sh
./shell_scripts/benchmark.sh
```
```
========== GPU RapidSnark standalone prove  ==========
/home/okxdex/data/zkdex-pap/services/rapidsnark/build_prover_gpu/src/prover ./build/jwt/jwt_single1.zkey ./build/jwt/witness.wtns ./build/jwt/proof.json ./build/jwt/public.json
mem 1800 MB
time 1.400000 s
cpu 364 
========== RapidSnark standalone prove  ==========
/home/okxdex/data/zkdex-pap/services/rapidsnark/build_prover/src/prover ./build/jwt/jwt_single1.zkey ./build/jwt/witness.wtns ./build/jwt/proof.json ./build/jwt/public.json
mem 2145 MB
time 1.623000 s
cpu 3755 
========== Should run proverServer in advance ==========
========== RapidSnark server prove  ==========
node /home/okxdex/data/zkdex-pap/services/rapidsnark/tools/request.js ./build/input_jwt_single1.json jwt_single1
mem 52 MB
time 2.151000 s
cpu 47 
========== SnarkJS prove  ==========
snarkjs groth16 prove ./build/jwt/jwt_single1.zkey ./build/jwt/witness.wtns ./build/jwt/proof.json ./build/jwt/public.json
mem 9000 MB
time 13.533000 s
cpu 567 
Proof size: 804
```

## on-chain verification

in our code, we have examples of verifying an OpenAI JWT on-chain. however, `./contracts/Blind.sol` is not updated with the current state of the circuit, since our proof of concept app, Nozee, does not use on-chain verification.

however, if you are interested in deploying on-chain, `./scripts/deploy.js` allows you to do a hardhat deploy, and `./test/blind.test.js` are examples of how we tested and deployed our previously working Blind.sol contract.

run hardhat contract tests, first create a `secret.json` file that has a private key and goerli node provider endpoint.
```
yarn test
```
