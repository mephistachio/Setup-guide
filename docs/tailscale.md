# Tailscale

The [Tailscale Kubernetes operator](https://tailscale.com/kb/1236/kubernetes-operator) is a subchart of [charts/homeserver](../charts/homeserver). It lets you reach services on the homeserver from your phone and laptop from anywhere, over WireGuard, with no port forwarding and no public exposure. It adds an ingress class `tailscale`; Traefik stays the default for `*.home.lan` on the LAN.

The operator needs OAuth credentials to register devices in your tailnet. Everything else is in the repo.

## 1. Tailnet setup (once, in the browser)

All in the [admin console](https://login.tailscale.com/admin).

1. **Account.** Sign up at tailscale.com (the free personal plan is enough).
2. **DNS.** Under *DNS*, enable **MagicDNS** and **HTTPS Certificates**. Services get a name like `whoami.<tailnet>.ts.net` with a valid certificate. Note your tailnet name (`<tailnet>.ts.net`) here.
3. **Tags.** Under *Access controls*, add to the policy file:

   ```json
   "tagOwners": {
     "tag:k8s-operator": [],
     "tag:k8s":          ["tag:k8s-operator"]
   }
   ```

   The operator is tagged `tag:k8s-operator`; the proxies it creates for each exposed service are `tag:k8s`. If you already have a `tagOwners` block, add the two entries to it.
4. **OAuth client.** Under *Settings → OAuth clients* (newer UIs: *Trust credentials*), generate a client with:
   - **Devices → Core**: Write, tag `tag:k8s-operator`
   - **Keys → Auth Keys**: Write, tag `tag:k8s-operator`

   Copy the **client ID** and **client secret**. The secret is shown **once**, so save it in your password manager right away (see below) before closing the page.

## 2. Store the credentials

The OAuth client goes into an encrypted file committed to the repo; the private key stays out of it. Follow [secrets.md](secrets.md) steps 1–4, with the values from step 1.4 above:

```yaml
tailscale-operator:
  oauth:
    clientId: ...
    clientSecret: ...
```

## 3. Deploy

```sh
ansible-playbook ansible/playbooks/site.yml -K --tags chart
```

The playbook decrypts `charts/homeserver/secrets.enc.yaml` and merges it into the Helm values, and warns if the file is missing. Without Ansible, see [secrets.md](secrets.md#5-everyday-use).

Check:

```sh
kubectl -n homeserver get pods            # operator Running
kubectl -n homeserver logs deploy/operator
```

A machine called `homelab-operator` appears under *Machines* in the admin console. If the pod crash-loops with an authentication error, the client ID/secret or the scopes/tags above are wrong.

## 4. Join your devices

Install Tailscale on the laptop and your Android phone and sign in with the same account. Android: the Tailscale app from Google Play, then toggle it on.

## 5. Expose a service

Add an Ingress with class `tailscale` (the host name is the first label of the tailnet name, with TLS):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami-tailscale
spec:
  ingressClassName: tailscale
  defaultBackend:
    service:
      name: homeserver-whoami
      port: {number: 80}
  tls:
    - hosts: [whoami]
```

After a minute, `https://whoami.<tailnet>.ts.net` works from any device on your tailnet. The operator starts one small proxy pod per exposed service. To expose a plain Service without an Ingress, annotate it with `tailscale.com/expose: "true"`.

Services stay private: only your tailnet can reach them. For public access you would add `tailscale.com/funnel: "true"` to the Ingress, which this setup does not do.

## 6. Passwords and secrets

| Secret | Who reads it | Where it lives |
|---|---|---|
| Tailscale account login, other web logins | you | Google Password Manager |
| OAuth client secret (the operator's credentials) | Helm, from this laptop | encrypted in `secrets.enc.yaml`, in git |
| age private key that decrypts the file above | this laptop | `~/.config/sops/age/keys.txt` + a Google Password Manager entry |

[Google Password Manager](https://passwords.google.com) autofills passwords and passkeys on Android and in Chrome with nothing to install. It has no API and no self-hosting, so it holds the human-facing logins and the **one** age key; everything else is in the repo, encrypted. See [secrets.md](secrets.md) for the setup, the restore test and the limits.

Also save the OAuth client secret as its own entry right when you create it: Tailscale shows it once.

**If you later want self-hosted:** [Vaultwarden](https://github.com/dani-garcia/vaultwarden) (Bitwarden-compatible) can be added as another subchart and exposed with a Tailscale Ingress. Android uses *one* autofill service at a time, so the Bitwarden app would *replace* Google Password Manager for autofill rather than integrate with it. Keep the age key backup outside the homeserver either way.

## 7. Remove or rotate

- **Rotate:** generate a new OAuth client, re-encrypt `secrets.enc.yaml` (`sops charts/homeserver/secrets.enc.yaml`), redeploy, then delete the old client in the admin console.
- **Disable:** set `tailscale-operator.enabled: false` in `values.yaml` and redeploy. Delete the leftover machines in the admin console.
