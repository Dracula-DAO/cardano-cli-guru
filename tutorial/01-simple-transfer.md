# **`cardano-cli` Exercise 01: Simple Transfer**
**Note**: you'll need your local node to be running and fully synced to complete this exercise. 
If you're using Cardano CLI Guru within the **[Jambhala](https://github.com/iburzynski/jambhala)** framework, the `j node` command will start `cardano-node` using whichever network is set by the `$CARDANO_NETWORK_ID` variable in `cardano-cli-guru/.env` (`preview` testnet by default).

Alternatively, if your node is correctly installed and configured for use with Cardano EZ-Installer, you can use the appropriate alias in your terminal to start `cardano-node`:
- `preprod-node` for preprod testnet
- `preview-node` for preview testnet

See the **[Cardano EZ-Installer README](https://github.com/iburzynski/cardano-ez-installer)** if you face any difficulty.

***
In this exercise we'll practice generating wallet addresses and submitting a simple transfer transaction.

The process consists of the following steps:
1. **[Generate keys and addresses](#generate)**
2. **[Fund wallet from faucet and select UTXO](#fund)**
3. **[Build the transaction](#build)**
4. **[Sign the transaction](#sign)**
5. **[Submit the transaction](#submit)**
6. **[Check the result](#check)**

***
## **1. <a id="generate"></a> Generate keys and addresses**
To build a transaction between two parties, we first need to generate key-pairs and addresses for the sender (`alice`) and recipient (`bob`). The `cardano-cli`'s `address` command has two subcommands to generate key pairs and derive the associated Cardano address: **`key-gen`** and **`build`**.

### **Generating keys**
The `key-gen` subcommand creates an asymmetric cryptographic **key-pair**, consisting of public and private components (**verification key** and **signing key**, respectively). 
* The **verification** key is used to derive an associated address on the network that can receive UTxOs and for verifying the owner's signatures in transactions.
* The **signing key** is used to provide signatures that authorize the spending of associated UTxOs.

>**Note:** for real user key-pairs intended for use on the Cardano mainnet, the **signing key** should never, ever be shared with anyone or stored digitally.

We must provide the command two filepaths as options, indicating where we'd like the associated files to be created (we'll save them to the `$KEYS_PATH` directory, which is `cardano-cli-guru/assets/keys` by default): 

```sh
cardano-cli address key-gen \
--verification-key-file $KEYS_PATH/alice.vkey \
--signing-key-file $KEYS_PATH/alice.skey
```

### **Deriving addresses**
Every possible verification key corresponds to an address on the Cardano network (whether that address has taken part in any on-chain transactions or not). Thus we can think of every possible Cardano address as always already existing in *virtual* form. But for the owner of a given key-pair to take part in a Cardano transaction, we need to know their address. While the subcommand we'll use is (perhaps misleadingly) called `build`, we should think of this operation not as *creating* an address (since it already exists *in potentia*), but as a function that takes the verification key and *derives* or *reveals* the address that is latent within it.

When we provide the verification key file to the `build` subcommand, it computes the address by hashing the verification key and encoding it in **[Bech32](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki)** format, which adds a "human readable part" based on the network specified by the `CARDANO_NODE_NETWORK_ID` environment variable (`addr` for mainnet, `addr_test` for `preprod` and `preview` testnets), followed by a separator character (`1`) and the encoded address:

```sh
cardano-cli address build \
--payment-verification-key-file $KEYS_PATH/alice.vkey \
--out-file $ADDR_PATH/alice.addr
```

A file containing the address will be created in the `$ADDR_PATH` directory (`cardano-cli-guru/assets/addr` by default), and we can view the address using the `cat` command:

```sh
cat $ADDR_PATH/alice.addr
```

The `addr` helper script also displays an address when applied to a name:

```sh
addr alice
```

We'll rely on this script frequently when building transactions, as it provides an easy way to interpolate a given address into a shell command (i.e. `$(addr alice)`).

>**Note:** we'll need to use this combination of subcommands to create additional key-pairs and addresses throughout the course of our tutorials. Cardano CLI Guru provides a helper script called `key-gen` to streamline this process, consisting of the following:

```sh
# cardano-cli-guru/scripts/key-gen

vkey="$KEYS_PATH/$1.vkey"
skey="$KEYS_PATH/$1.skey"
addr="$ADDR_PATH/$1.addr"

...

cardano-cli address key-gen \
--verification-key-file $vkey \
--signing-key-file $skey

cardano-cli address build \
--payment-verification-key-file $vkey \
--out-file $addr
```

The script first assigns variables for the three output files it will produce (`vkey`, `skey` and `addr`) using the `KEYS_PATH` and `ADDR_PATH` filepaths configured in our `.envrc` file and the name supplied as argument to the script (`$1`).

It then runs the `key-gen` subcommand, providing out-filepaths for the `--verification-key-file` and `--signing-key-file` parameters.

After generating the key files, the script runs the `build` subcommand, providing the location of the `.vkey` file to the `--payment-verification-key-file` parameter, as well as the desired `--out-file` location.

### **Run the `key-gen` script**
Run the `key-gen` script to create a keypair and derive an address `bob`:

```sh
$ key-gen bob
wrote verification key to: assets/keys/bob.vkey
wrote signing key to: assets/keys/bob.skey
wrote address to: assets/keys/bob.addr
```

### **Viewing a User's Address**
When using `cardano-cli`, we'll frequently need to reference user addresses (stored in `.addr` files). Cardano CLI Guru provides a simple script that takes a user's name as argument and outputs their address:

```sh
# cardano-cli-guru/scripts/addr

echo $(cat "$ADDR_PATH/$1.addr")
```

We can run this script to view a user's address in the terminal, and later to interpolate addresses into `cardano-cli` commands.

Run the script for `alice` to see her address:

```sh
$ addr alice
addr_test1vqa9qw00et75eqkfz6dnttaj0gtkw0mw67z75zr38xn95hgvh8v80
```

***
## **2. <a id="fund"></a> Fund wallet from faucet and select UTXO**
We now have addresses for our transaction parties, but they don't contain any funds to make transactions with.

Copy Alice's wallet address from the previous step and fund the wallet with 10,000 Test ADA from the [Testnet Faucet](https://docs.cardano.org/cardano-testnet/tools/faucet).

>Make sure to select the correct `Environment` at the Testnet Faucet. By default Cardano EZ-Installer uses the `Preview Testnet`.

>Make sure to use the address generated for **your** Alice wallet, not the sample output provided above - otherwise you'll fund the wrong Alice!

Let's check the UTXOs at Alice's address to confirm receipt of the Test ADA. The **`utxo`** subcommand of `cardano-cli`'s `query` command is used to list the UTXOs at an address. As this is a very common operation, Cardano CLI Guru provides a script called `utxos` that contains the following:

```sh
# cardano-cli-guru/scripts/utxos

cardano-cli query utxo \
--address $(addr $1)
```

The script runs the `query utxo` command, interpolating the address of the user name we supply the script as argument (using the `addr` script).

Run the script with `alice` as argument:

```sh
$ utxos alice
                           TxHash                                 TxIx        Amount
--------------------------------------------------------------------------------------
9c1203e214524e72f0e686b09821dc4c46a2b3cf0c6ee312f52a97cde00d1765     0        10000000000 lovelace + TxOutDatumNone
```

Copy the `TxHash` value from your terminal output and create a temporary variable called `U` to store this value, followed by a `#` symbol and the `TxIx` value:

```sh
$ U=9c1203e214524e72f0e686b09821dc4c46a2b3cf0c6ee312f52a97cde00d1765#0
```

***
## **3. <a id="build"></a> Build the transaction**
`cardano-cli`'s `transaction` command has two subcommands for building transactions: `build` and `build-raw`.

### **`build` vs. `build-raw`**
The **`build`** subcommand is a higher-level interface that builds an automatically **balanced** transaction.
* A Cardano transaction is **balanced** when the value of its **input** UTXOs equals the value its **output** UTXOs plus the transaction **fee** (`inputs = outputs + fee`).

* The `build` subcommand automatically calculates the transaction fee and deducts it from the transaction output, creating a balanced transaction.

* The `build` subcommand also doesn't require us to manually specify a **validity interval** for the transaction (a window of time in which the transaction is considered valid, specified in terms of slots). We'll learn more about validity intervals in the next exercise.

The **`build-raw`** subcommand is a lower-level interface for building transactions, which requires us to manually calculate the transaction's fee and specify its validity interval. It is less convenient to use, but provides us deeper insight into how transactions are constructed, so we'll practice using it in the next exercise.

### **Building transactions with `transaction build`**
We'll use the `build` subcommand to build a simple transaction that transfers 250 ADA from Alice to Bob:

```sh
cardano-cli transaction build \
--tx-in $U \
--tx-out $(addr bob)+250000000 \
--change-address $(addr alice) \
--out-file $TX_PATH/transfer.raw
```

Note the various parameters that must be provided:
* **Input(s)**: the `--tx-in` parameter specifies funds that will be spent by the transaction, which are outputs (UTXOs) from earlier transactions. A transaction can have multiple inputs, in which case we'd include multiple `--tx-in` lines with an associated UTXO. Here we are spending a single input, and its UTXO is the one from Alice's wallet we selected and stored in the variable `U`.
* **Output(s)**: the `--tx-out` parameter specifies where funds (UTXOs) will be sent. An output consists of a payment address and an amount. A transaction can have multiple outputs, in which case we'd include multiple `--tx-out` lines with associated addresses and amounts. Here we are sending a single output to Bob, whose address we include by interpolating the output of the `addr` script with `bob` as argument. We then specify the amount of the transfer (`+250000000`, or 250 ADA).
* **Change Address**: the `--change-address` parameter specifies which address the balance of funds will be sent to after the outputs and fees are deducted from the inputs. Since the input UTXOs must be spent in their entirety, any excess funds must be sent as an additional transaction output to some address (in this case Alice, who the input belongs to).
* **Out File**: the `--out-file` parameter specifies the file path where the transaction data will be stored. The data is stored in `json` format, but its file extension doesn't matter (by convention `.raw` is used). The filename we choose (here `transfer`) can be passed as an argument to helper scripts that Cardano CLI Guru provides for convenience.

When we run this command, we'll see terminal output that includes the automatically calculated fee:

```sh
Estimated transaction fee: Lovelace 165721
```

If we inspect the `transfer.raw` file created in the `assets/tx` directory, we see its contents are a `json` object with three attributes: `type`, `description`, and `cborHex`. This contains all the transaction information required for the next steps of the submission process.

```json
{
    "type": "Unwitnessed Tx BabbageEra",
    "description": "Ledger Cddl Format",
    "cborHex": "84a300818258209c1203e214524e72f0e686b09821dc4c46a2b3cf0c6ee312f52a97cde00d1765000182a200581d60603bc829fd1b4bfaff4d6d17193968a21a8899ab816c31af296f8628011a0ee6b280a200581d603a5039efcafd4c82c9169b35afb27a17673f6ed785ea087139a65a5d011b000000024522aa27021a00028759a0f5f6"
}
```

***
## **4. <a id="sign"></a> Sign the transaction**
The next step is for Alice to sign the transaction with her secret key. For this we use the **`sign`** subcommand of `cardano-cli`'s `transaction` command.

```sh
cardano-cli transaction sign \
--tx-body-file $TX_PATH/transfer.raw \
--signing-key-file $KEYS_PATH/alice.skey \
--out-file $TX_PATH/transfer.signed
```

>**Note:** since we'll use this command multiple times over the course of these exercises with similar arguments, Cardano CLI Guru provides a helper script called `tx-sign`, consisting of the following:

```sh
# cardano-cli-guru/scripts/tx-sign

tx="$TX_PATH/$1"
skeyfiles=""

for user in "${@:2}"; do
  skeyfiles+="--signing-key-file $KEYS_PATH/$user.skey "
done

cardano-cli transaction sign \
--tx-body-file "$tx.raw" \
$skeyfiles \
--out-file "$tx.signed"
```

The script first assigns the variable `tx` using the transaction filepath location configured in our `.envrc` file, and the first (`$1`) argument to the script (the transaction name).

It then assigns a variable `skeyfiles` to an empty string, and loops over the remaining (`user`) arguments (`@:2`), appending a `--signing-key-file` entry with the user's secret key file. This allows us to provide multiple signers to the script if needed (we'll see this in a later exercise).

It then runs the `transaction sign` command, providing the `.raw` file location to the `--tx-body-file` option, the `skeyfiles` string produced by the loop, the network option, and the filepath/name for the out-file.

Run the script by providing the transaction name (i.e. `transfer`) and signer name (i.e. `alice`) as arguments:

```sh
tx-sign transfer alice
```

Compare the contents of the out-file created at `assets/tx/transfer.signed` with the `.raw` file from the previous step:

```json
{
    "type": "Witnessed Tx BabbageEra",
    "description": "Ledger Cddl Format",
    "cborHex": "84a300818258209c1203e214524e72f0e686b09821dc4c46a2b3cf0c6ee312f52a97cde00d1765000182a200581d60603bc829fd1b4bfaff4d6d17193968a21a8899ab816c31af296f8628011a0ee6b280a200581d603a5039efcafd4c82c9169b35afb27a17673f6ed785ea087139a65a5d011b000000024522aa27021a00028759a1008182582074937754b5316e3b75b8371de51142ecaaf467945da8d744e4db0447e11f7e47584003b699163a41ba18f41c1fedb3eee7909032e5759cee79300b37dfe94d940f6de368ee1e102cbc331d4da9c204632f0c21d5d9c640c2300a92366c28da3c8909f5f6"
}
```

Notice the value of the `type` attribute has changed from `Unwitnessed Tx BabbageEra` to `Witnessed Tx BabbageEra` (signing a transaction is also referred to as "witnessing" it). The `cborHex` value has also increased in size, as it now includes Alice's signature.

***
## **5. <a id="submit"></a> Submit the transaction**
The final step of the transaction process is to submit the signed transaction. For this we use the **`submit`** subcommand of `cardano-cli`'s `transaction` command.

The `submit` subcommand is quite simple -- we just provide the filepath of the signed transaction:

```sh
cardano-cli transaction submit \
--tx-file $TX_PATH/transfer.signed
```

>**Note:** like `transaction sign`, we'll use `transaction submit` multiple times over the course of these exercises with similar arguments, so Cardano CLI Guru provides a helper script called `tx-submit`, consisting of the following:

```sh
# cardano-cli-guru/scripts/tx-submit

cardano-cli transaction submit \
--tx-file "$TX_PATH/$1.signed"
```

Run the script by providing the transaction name (`transfer`) as argument:

```sh
tx-submit transfer
Transaction successfully submitted.
```

🥳 ***Congratulations: you've submitted your first Cardano transaction!*** 🎉

***
## **6. <a id="check"></a> Check the result**
After waiting a minute, run the `utxos` script again, this time with `bob` as argument, to confirm the funds were received:

```sh
utxos bob
                           TxHash                                 TxIx        Amount
--------------------------------------------------------------------------------------
215650738a6a1c3fd8495075091a5853d9d48f9c818d791bdf464e6b8ae674c7     0        250000000 lovelace + TxOutDatumNone
```

You can copy the `TxHash` value and search for it on **[preview.cardanoscan.io](https://preview.cardanoscan.io/)** to view the transaction details (use **[preprod.cardanoscan.io](https://preprod.cardanoscan.io)** instead if using the `preprod` testnet).