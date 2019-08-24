blog


git remote add -f ananke https://github.com/budparr/gohugo-theme-ananke.git

git subtree add --prefix src/hugo/themes/ananke ananke master --squash

git fetch ananke master
git subtree pull --prefix src/hugo/themes/ananke ananke master --squash


