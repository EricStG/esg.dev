This is the code behind https://esg.dev

![Build status](https://github.com/EricStG/esg.dev/workflows/gh-pages/badge.svg)

Adding the remote (once per clone)
```
git remote add -f ananke https://github.com/theNewDynamic/gohugo-theme-ananke.git
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
