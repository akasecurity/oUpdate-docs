# oUpdate docs

Source of https://akasecurity.github.io/oUpdate-docs/, the documentation site for [oUpdate](https://github.com/akasecurity/oUpdate), an open-source package firewall that reads what each new release changed before it lands.

Build locally:

```
pip install --require-hashes -r requirements.txt
python3 -m mkdocs build --strict
```

Every push to `main` builds the site in strict mode and publishes it to GitHub Pages. Pull requests only build.

`requirements.txt` is pinned with hashes (`requirements.in` lists the two direct dependencies; regenerate with `uv pip compile requirements.in --universal --generate-hashes -o requirements.txt`).
