This is the code behind https://esg.dev

![Azure Static Web Apps CI/CD](https://github.com/EricStG/esg.dev/workflows/Azure%20Static%20Web%20Apps%20CI/CD/badge.svg)

Adding the remote (once per clone)
```
git remote add -f ananke https://github.com/budparr/gohugo-theme-ananke.git
```

Adding the subtree (once per repo)
```
git subtree add --prefix src/hugo/themes/ananke ananke master --squash
```

Updating the subtree
```
git fetch ananke master
git subtree pull --prefix src/hugo/themes/ananke ananke master --squash
```
