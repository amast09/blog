# Aaron Mast's Personal Blog 
Because every developer deserves one.

## Install Hugo
[https://github.com/gohugoio/hugo/releases](https://github.com/gohugoio/hugo/releases)

## Install AWS CLI
```bash
pip install awscli --upgrade --user
# Add AWS to Path
echo "export PATH=~/.local/bin:\$PATH" >> ~/.bashrc
source ~/.bashrc
```
## Configure AWS User
```bash
aws configure
```

## Initialize Hugo Theme
```bash
git submodule init
git pull --recurse-submodules
```

## Start hugo
```bash
hugo server -D
```

## Create a new post
```bash
hugo new posts/my-first-post.md

```

## Publish post
```bash
sh deploy-site.sh
```
