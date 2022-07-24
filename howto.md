# Howto work with the blog

May need to install some starter things.
```bash
sudo apt install ruby ruby-dev
gem install jekyll bundler
bundle install
```

Start a local webserver.
```bash
bundle exec jekyll build
```

Build the blog
```bash
bundle exec jekyll build -d docs
cp CNAME docs/
```
