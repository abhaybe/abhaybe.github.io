# Pointing a Namecheap domain at GitHub Pages

Replace `example.com` below with your domain. Do this after the site is live at https://abhaybe.github.io.

## 1. Verify the domain with GitHub (recommended, prevents takeover)

1. GitHub → your profile picture → **Settings** → **Pages** → **Add a domain** → enter `example.com`.
2. GitHub shows a TXT record like `_github-pages-challenge-abhaybe` with a value. Add it in Namecheap (step 2), then click **Verify**.

## 2. DNS records in Namecheap

Namecheap → **Domain List** → **Manage** next to the domain → **Advanced DNS**.

First delete the default parking records (the `CNAME www → parkingpage.namecheap.com` and any `URL Redirect` for `@`). Then add:

| Type | Host | Value |
|---|---|---|
| A Record | `@` | `185.199.108.153` |
| A Record | `@` | `185.199.109.153` |
| A Record | `@` | `185.199.110.153` |
| A Record | `@` | `185.199.111.153` |
| AAAA Record | `@` | `2606:50c0:8000::153` |
| AAAA Record | `@` | `2606:50c0:8001::153` |
| AAAA Record | `@` | `2606:50c0:8002::153` |
| AAAA Record | `@` | `2606:50c0:8003::153` |
| CNAME Record | `www` | `abhaybe.github.io.` |
| TXT Record | `_github-pages-challenge-abhaybe` | *(value from step 1)* |

TTL: Automatic is fine. Changes usually show up within 30 minutes (sometimes a few hours). Check with:

```bash
dig example.com +noall +answer
dig www.example.com +noall +answer
```

## 3. Tell GitHub Pages about the domain

1. Repo `abhaybe/abhaybe.github.io` → **Settings** → **Pages** → **Custom domain** → enter `example.com` → **Save**. This commits a `CNAME` file containing `example.com` to the repo (or add that one-line file yourself).
2. Pull that commit locally (`git pull`) so you don't overwrite it later.
3. In `_config.yml`, change `url:` to `https://example.com` and push. Links in the feed and page metadata use it.

`abhaybe.github.io` and `www.example.com` will both redirect to `example.com` automatically.

## 4. HTTPS

Once the DNS check in **Settings → Pages** turns green, GitHub requests a Let's Encrypt certificate (usually minutes, up to 24 hours). Then tick **Enforce HTTPS**. If the box stays greyed out for over a day, remove and re-save the custom domain to retrigger it.
