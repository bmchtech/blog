
# blog

bmch web

## dev

grab submodules:
```
git submodule update --init --recursive
```

```sh
hugo serve
```

this will run a live-reloaded dev server of hugo.

## dev: redbuild shell

```sh
# build container and launch shell
redbuild shell --crun-args "-p 1313:1313"
# run hugo serve inside container
hugo serve --bind 0.0.0.0
```

## build

```sh
hugo
```

this will output the built site in `public/`.
