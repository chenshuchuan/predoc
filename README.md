Predoc
======

Predoc is a Ruby on Rails web service for in-browser document previews. It converts supported documents — Microsoft Word, Excel, and PowerPoint — into PDFs for web browsers to display.

Installation
------------

### Dependencies

Predoc uses [LibreOffice](http://www.libreoffice.org/) (via [Docsplit](http://documentcloud.github.com/docsplit/)) to convert documents. Install LibreOffice on the web server using aptitude, apt-get, or yum:

    $ aptitude install libreoffice

centos7:   
    yum install libreoffice

乱码
```
centos7:
    cp simsun.ttc /usr/share/fonts
    yum install mkfontscale
    mkfontscale
    mkfontdir
    fc-cache

ubuntu:
    sudo cp simsun.ttc /usr/share/fonts
    sudo chmod 644 simsun.ttc
    sudo fc-cache -fv
```

Ruby版本
Ruby >~ 1.9.3:
亲自测试过: 2.4.0
[rvm install ruby](https://ruby-china.org/wiki/rvm-guide)

bundler版本:
bundler >~ 1.13.3
亲自测试过:

    gem install bundler --version 1.13.3

Configuration
-------------

You need to specify the paths for Predoc to create temporary and cache files. Create `predoc.rb` from the template file `config/initializers/predoc.rb.default` at the same location, and modify these variables:

* `WORKING_DIRECTORY` is where temporary files are downloaded and stored.
* `CACHE_ROOT_DIRECTORY` is where converted files are cached.

Make sure these directories are writable by the web server.

In addition, you can configure the maximum number of seconds allowed for a single conversion by changing `CONVERSION_TIMEOUT`.

### Conversion Statistics

If you use [StatsD](https://github.com/etsy/statsd/) to keep track of statistics, you can enable it by setting `STATSD_HOST`. Also configure `STATSD_PORT` and `STATSD_NAMESPACE` as needed. The application will increment one or more of the following stats related to requests for preview:

* General
  * `request` — when a preview is requested
  * `convert` — when a conversion process begins
  * `converted` — when a conversion process ends (sent as duration in milliseconds)
* Preview request successful
  * `sent.cached` — when a cached preview is available and sent
  * `sent.passthru` — when a PDF source file is sent directly without conversion
  * `sent.converted` — when a document is converted and sent
* Error handled internally
  * `rescue.inconvertible` — when an error occurred during a conversion process
  * `rescue.timeout` — when a conversion took too long and is aborted
* Preview request failed
  * `error.unreadable` — when a source file is not found or cannot be read
  * `error.unsupported` — when a source file type is not supported
  * `error.incomplete` — when a conversion yielded nothing due to some errors

Deployment
----------

You can use Capistrano to deploy Predoc. Create `deploy.rb` from the template file `config/deploy.rb.example` at the same location. Customize settings as needed. You have the option to deploy to multiple stages (e.g. staging, production, etc.) In that case, you'll need to create `config/deploy/{stage_name}.rb` as well.

Usage
-----

To create a preview, point your web browser to the web service:

    http://example.com/viewer?url=http://path/to/document

*Note: Replace `example.com` with the server hostname, and `http://path/to/document` with an actual web-hosted document*

Testing
-------

To run unit tests, first create `test.rb` from the template file `config/test.rb.default` at the same location. Change the URL values in `FIXTURE_URLS` to real web-hosted test documents (not provided). Run this command to start the test:

    ruby -Itest test/functional/documents_controller_test.rb



Note
-------

    cp config/initializers/predoc.rb.default config/initializers/predoc.rb

    bundle install --path vendor/bundle

    bundle exec rake assets:precompile

    bundle exec puma -p 8600

nginx.conf:

```
    location /viewer/ {
        proxy_pass http://127.0.0.1:9000;
        proxy_connect_timeout 600s;
        proxy_send_timeout 600s;
        proxy_read_timeout 600s        
    }

    location /convert/ {
        proxy_pass http://127.0.0.1:9000;
        proxy_connect_timeout 600s;
        proxy_send_timeout 600s;
        proxy_read_timeout 600s        
    }

    location /assets/ {
        proxy_pass http://127.0.0.1:9000;
    }

```

predoc.conf
```
server {
    listen 9000;
    server_name localhost;
    root /path/predoc/public;
    location / {
        proxy_pass http://127.0.0.1:8600;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $http_host;
        #proxy_set_header X-Forwarded-Proto https;
        proxy_redirect default;
        proxy_connect_timeout 600s;
        proxy_send_timeout 600s;
        proxy_read_timeout 600s;        
        proxy_buffer_size       32k;
        proxy_buffers           32 256k;
        proxy_busy_buffers_size 512k;
        proxy_temp_file_write_size 512k;
    }
}
```