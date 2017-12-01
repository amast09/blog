# Aaron Mast's Personal Blog 
Because every developer deserves one.

## Install Hugo
[https://github.com/gohugoio/hugo/releases](https://github.com/gohugoio/hugo/releases)

## Install AWS CLI and Add it to Our Path
```bash
$ pip install awscli --upgrade --user
$ echo "export PATH=~/.local/bin:\$PATH" >> ~/.bashrc
$ source ~/.bashrc
```
## Configure AWS User
```bash
$ aws configure
```

## Initialize Hugo Theme
```bash
$ git submodule init
$ git pull --recurse-submodules
```

## Start hugo
```bash
$ hugo server -D
```

## Create a new post
```bash
$ hugo new blog/new-post.md
```

## Publish post
```bash
$ sh deploy-site.sh
```
