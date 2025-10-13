### Who this is for

Federation node operators

### Prerequisite
* Get [btc-utils](https://storage.googleapis.com/internal-binaries/btc-utils)

### Export (Backup) Btc-Server key

**Locate DB** e.g `mainnet/signing-server/db`

**Run export**
```bash
btc-utils export-key-package \
--db="<PATH_TO_BTC_SERVER_DB>" \
--output="exported_key_package.bin"
```

**You will be prompted**
```
Passphrase:
Confirm passphrase:
```

Expect: Successfully exported encrypted key package to `exported_key_package.bin`

**Secure**

* Change file permissions:
    ```bash
    chmod 600 exported_key_package.bin
    ```
* Move to a secure store (Secrets Manager, Vault, KMS-encrypted storage etc). Keep the passphrase in a different secret.

* If the store only accepts text, encode first:
    ```bash
    base64 exported_key_package.bin > exported_key_package.b64
    ```

### Import (Restore) Btc-Server key

> Use this to restore on a new host or to test your backup. You need the encrypted file and the passphrase you set at export time.

Choose a target DB directory (empty or existing): e.g `mainnet/signing-server-restore/db`

**Import**
```bash
btc-utils import-key-package \
--db="<TARGET_DB_DIR>" \
--input="<PATH_TO_EXPORTED_BIN>"
```
* If you see: `Existing key package may not be overwritten`, re-run with:
    ```bash
    btc-utils import-key-package \
    --db="<TARGET_DB_DIR>" \
    --input="<PATH_TO_EXPORTED_BIN>" \
    --force-overwrite
    ```

Command should report: `Successfully imported decrypted key package.`

### Operational Guidance

* Never store the export file and its passphrase in the same secret

* Keep local copies ephemeral: upload to your secure store, verify, then securely delete the local file.

* To rotate the passphrase: re-export the same key package with a new passphrase and replace the stored artifact; the underlying key package stays the same, only its encryption changes.