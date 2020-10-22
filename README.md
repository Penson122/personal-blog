# personal-blog

Not another dev blog. It's available over at https://jackpenson.dev.

Requirements:
* [docker](https://docs.docker.com/get-docker/)
* [rvm](https://rvm.io/rvm/install)
* `gem install bundle jekyll`

Getting Started:
* `rvm use`
* `bundle install`
* `bundle exec jekyll serve` - watch for file changes and view blog.

Building:
* `bundle exec jekyll build` - this will generate the static site
* `docker build -t blog .` - Copy the static site into the nginx container

View the containerised site locally:
* `docker run -p 8080:80 blog`
