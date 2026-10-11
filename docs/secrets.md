# Secrets with SOPS and age

Secrets that Helm needs (today: the Tailscale OAuth client) are committed to git **encrypted**, with [SOPS](https://github.com/getsops/sops) and [age](https://github.com/FiloSottile/age). One age private key decrypts them. That key is kept in two places: on this laptop, and in Google Password Manager as the backup.

| What | Where | Committed |
|---|---|---|
| age **public** key | [.sops.yaml](../.sops.yaml) | yes |
| encrypted values | `charts/homeserver/secrets.enc.yaml` | yes |
| age **private** key | `~/.config/sops/age/keys.txt` + Google Password Manager | **never** |
| plaintext values (temporary) | `charts/homeserver/secrets.yaml` | no, gitignored |

SOPS encrypts only the values; key names stay readable, so diffs show *what* changed.

Run every command below from the repo root.

## 1. Install the tools

```sh
ansible-playbook ansible/playbooks/site.yml -K --tags sops
```

This installs `age` (apt) and `sops` (GitHub release, checksum verified). `ansible-galaxy collection install -r ansible/requirements.yml` provides the `community.sops` collection the deploy uses.

## 2. Create the key

```sh
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
chmod 600 ~/.config/sops/age/keys.txt
```

`age-keygen` prints `Public key: age1...`. Put that into [.sops.yaml](../.sops.yaml), replacing `REPLACE_WITH_YOUR_AGE_PUBLIC_KEY`, and commit the change. A public key is safe to commit.

## 3. Back up the private key in Google Password Manager

Do this **before** encrypting anything.

1. Show only the secret line, and copy it:

   ```sh
   grep AGE-SECRET-KEY ~/.config/sops/age/keys.txt
   ```

   It is one line starting with `AGE-SECRET-KEY-1`, about 74 characters.
2. Open [passwords.google.com](https://passwords.google.com) (or Chrome → Password Manager) → **Add password**:
   - Site: `homelab.home.lan`
   - Username: `sops-age-private-key`
   - Password: the line you copied
3. Check it synced: the entry appears on the Android phone under *Settings → Passwords & accounts → Google Password Manager*.
4. Clear the terminal (`clear`) and the clipboard.

**Test the restore once** while you still have the original: move `keys.txt` aside, paste the line from Google Password Manager into a new `keys.txt` (the file may contain just that line), `chmod 600` it, and run `sops -d charts/homeserver/secrets.enc.yaml` once the file exists (step 4). If it prints the values, the backup works; delete the moved-aside copy.

To use the repo on another machine, put the same line in `~/.config/sops/age/keys.txt` there.

## 4. Create the encrypted secrets

```sh
cp charts/homeserver/secrets.example.yaml charts/homeserver/secrets.yaml
$EDITOR charts/homeserver/secrets.yaml                       # fill in the values
sops --encrypt charts/homeserver/secrets.yaml > charts/homeserver/secrets.enc.yaml
shred -u charts/homeserver/secrets.yaml
git add .sops.yaml charts/homeserver/secrets.enc.yaml
```

The plaintext `secrets.yaml` is gitignored as a safety net, but delete it anyway. Check that no real value appears in `secrets.enc.yaml` (every value should read `ENC[AES256_GCM,...]`) before committing.

## 5. Everyday use

```sh
sops charts/homeserver/secrets.enc.yaml        # edit: opens $EDITOR with decrypted text, re-encrypts on save
sops -d charts/homeserver/secrets.enc.yaml     # print decrypted
ansible-playbook ansible/playbooks/site.yml -K --tags chart   # deploy, decrypts automatically
```

Without Ansible:

```sh
sops -d charts/homeserver/secrets.enc.yaml | \
  helm upgrade --install homeserver charts/homeserver -n homeserver -f -
```

The deploy task runs with `no_log`, because the Helm module prints the release values. If a deploy fails, its error message is hidden; run the `helm` command above to see it.

## 6. Rotating the key

Generate a new key, add its public key to `.sops.yaml` (keep the old one until done), run `sops updatekeys charts/homeserver/secrets.enc.yaml`, then remove the old key from `.sops.yaml`, run `updatekeys` again, and replace the Google Password Manager entry. Anyone who had the old key can still read older commits, so also rotate the underlying secrets (a new Tailscale OAuth client) if the old key leaked.

## 7. What this does and doesn't protect

- Protects the secrets in git: a copy of the repo, even public, does not reveal them without the age key.
- Your Google account is now the root of trust for the backup. Keep 2-step verification on, and a screen lock on the phone.
- If you lose **both** the laptop and the Google entry, the encrypted file is unreadable. That is recoverable here: create a new Tailscale OAuth client and encrypt a new file.
- Decrypted values still reach the cluster: Helm stores release values, and the operator's `operator-oauth` Secret holds the credentials, as base64 in k3s's datastore on the root disk. Anyone with root on the server can read them.
