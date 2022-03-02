# README

## Steps

```s
# Create project
$ rails new demo-rack-attack
$ cd demo-rack-attack

# Add in gems
$ bundler add rack-attack

# Create a test controller
$ bin/rails g controller hello index
      create  app/controllers/hello_controller.rb
       route  get 'hello/index'
      invoke  erb
      create    app/views/hello
      create    app/views/hello/index.html.erb
      invoke  test_unit
      create    test/controllers/hello_controller_test.rb
      invoke  helper
      create    app/helpers/hello_helper.rb
      invoke    test_unit
```

Update `config/application.rb`:

```rb
require_relative "boot"

require "rails/all"

# Require the gems listed in Gemfile, including any gems
# you've limited to :test, :development, or :production.
Bundler.require(*Rails.groups)

module DemoRackAttack
  class Application < Rails::Application
    # Initialize configuration defaults for originally generated Rails version.
    config.load_defaults 7.0

    # Configuration for the application, engines, and railties goes here.
    #
    # These settings can be overridden in specific environments using the files
    # in config/environments, which are processed later.
    #
    # config.time_zone = "Central Time (US & Canada)"
    # config.eager_load_paths << Rails.root.join("extras")
    config.action_controller.default_protect_from_forgery = false if ENV['RAILS_ENV'] == 'development'
  end
end
```

Inside of `config/environments/development.rb`, update `config.cache_store` to be:

```rb
config.cache_store = :redis_cache_store, { url: ENV.fetch('REDIS_URL', 'redis://localhost:6379/1') }
```

Update `config/initializers/rack_attack.rb`:

```rb
class Rack::Attack

  ### Configure Cache ###

  # If you don't want to use Rails.cache (Rack::Attack's default), then
  # configure it here.
  #
  # Note: The store is only used for throttling (not blocklisting and
  # safelisting). It must implement .increment and .write like
  # ActiveSupport::Cache::Store

  # Rack::Attack.cache.store = ActiveSupport::Cache::MemoryStore.new

  ### Throttle Spammy Clients ###

  # If any single client IP is making tons of requests, then they're
  # probably malicious or a poorly-configured scraper. Either way, they
  # don't deserve to hog all of the app server's CPU. Cut them off!
  #
  # Note: If you're serving assets through rack, those requests may be
  # counted by rack-attack and this throttle may be activated too
  # quickly. If so, enable the condition to exclude them from tracking.

  # Throttle all requests by IP (60rpm)
  #
  # Key: "rack::attack:#{Time.now.to_i/:period}:req/ip:#{req.ip}"
  throttle('req/ip', limit: 300, period: 5.minutes) do |req|
    req.ip # unless req.path.start_with?('/assets')
  end

  ### Prevent Brute-Force Login Attacks ###

  # The most common brute-force login attack is a brute-force password
  # attack where an attacker simply tries a large number of emails and
  # passwords to see if any credentials match.
  #
  # Another common method of attack is to use a swarm of computers with
  # different IPs to try brute-forcing a password for a specific account.

  # Throttle POST requests to /login by IP address
  #
  # Key: "rack::attack:#{Time.now.to_i/:period}:logins/ip:#{req.ip}"
  throttle('logins/ip', limit: 5, period: 20.seconds) do |req|
    if req.path == '/login' && req.post?
      req.ip
    end
  end

  # Throttle POST requests to /login by email param
  #
  # Key: "rack::attack:#{Time.now.to_i/:period}:logins/email:#{normalized_email}"
  #
  # Note: This creates a problem where a malicious user could intentionally
  # throttle logins for another user and force their login requests to be
  # denied, but that's not very common and shouldn't happen to you. (Knock
  # on wood!)
  throttle('logins/email', limit: 5, period: 20.seconds) do |req|
    if req.path == '/login' && req.post?
      # Normalize the email, using the same logic as your authentication process, to
      # protect against rate limit bypasses. Return the normalized email if present, nil otherwise.
      req.params['email'].to_s.downcase.gsub(/\s+/, "").presence
    end
  end

  ### Custom Throttle Response ###

  # By default, Rack::Attack returns an HTTP 429 for throttled responses,
  # which is just fine.
  #
  # If you want to return 503 so that the attacker might be fooled into
  # believing that they've successfully broken your app (or you just want to
  # customize the response), then uncomment these lines.
  # self.throttled_response = lambda do |env|
  #  [ 503,  # status
  #    {},   # headers
  #    ['']] # body
  # end

  throttle('example', limit: 5, period: 10.seconds) do |req|
    if req.path == '/hello' && req.get?
      req.ip
    end
  end
end
```

In `app/controllers/hello_controller.rb`:

```rb
class HelloController < ApplicationController
  def index
    render json: { message: 'Hello World' }
  end
end
```

In `routes.rb`:

```rb
Rails.application.routes.draw do
  resources :hello, only: [:index]
  # Define your application routes per the DSL in https://guides.rubyonrails.org/routing.html

  # Defines the root path route ("/")
  # root "articles#index"
end
```

Ensure you turn on the cache before you run the server:

```s
# Turn on cache and run server
$ bin/rails dev:cache
$ bin/rails s

# Use ab to run 10 requests in quick succession.
# Note that 5 fail due to throttling.
$ ab -n 10 http://localhost:3000/hello
This is ApacheBench, Version 2.3 <$Revision: 1879490 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient).....done


Server Software:
Server Hostname:        localhost
Server Port:            3000

Document Path:          /hello
Document Length:        25 bytes

Concurrency Level:      1
Time taken for tests:   0.190 seconds
Complete requests:      10
Failed requests:        5
   (Connect: 0, Receive: 0, Length: 5, Exceptions: 0)
Non-2xx responses:      5
Total transferred:      5180 bytes
HTML transferred:       185 bytes
Requests per second:    52.62 [#/sec] (mean)
Time per request:       19.005 [ms] (mean)
Time per request:       19.005 [ms] (mean, across all concurrent requests)
Transfer rate:          26.62 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.1      0       0
Processing:    13   19   5.9     18      32
Waiting:       12   19   5.9     17      32
Total:         13   19   5.9     18      33

Percentage of the requests served within a certain time (ms)
  50%     18
  66%     20
  75%     21
  80%     24
  90%     33
  95%     33
  98%     33
  99%     33
 100%     33 (longest request)
```

Logs from Rails server:

```s
Started GET "/hello" for ::1 at 2022-03-02 21:58:42 +1000
Processing by HelloController#index as */*
Completed 200 OK in 12ms (Views: 0.5ms | ActiveRecord: 0.0ms | Allocations: 2367)


Started GET "/hello" for ::1 at 2022-03-02 21:58:42 +1000
Processing by HelloController#index as */*
Completed 200 OK in 1ms (Views: 0.2ms | ActiveRecord: 0.0ms | Allocations: 114)


Started GET "/hello" for ::1 at 2022-03-02 21:58:42 +1000
Processing by HelloController#index as */*
Completed 200 OK in 1ms (Views: 0.3ms | ActiveRecord: 0.0ms | Allocations: 114)


Started GET "/hello" for ::1 at 2022-03-02 21:58:42 +1000
Processing by HelloController#index as */*
Completed 200 OK in 1ms (Views: 0.3ms | ActiveRecord: 0.0ms | Allocations: 114)


Started GET "/hello" for ::1 at 2022-03-02 21:58:42 +1000
Processing by HelloController#index as */*
Completed 200 OK in 1ms (Views: 0.2ms | ActiveRecord: 0.0ms | Allocations: 114)


Started GET "/hello" for ::1 at 2022-03-02 21:58:42 +1000
Started GET "/hello" for ::1 at 2022-03-02 21:58:42 +1000
Started GET "/hello" for ::1 at 2022-03-02 21:58:42 +1000
Started GET "/hello" for ::1 at 2022-03-02 21:58:42 +1000
Started GET "/hello" for ::1 at 2022-03-02 21:58:42 +1000
```

The last 5 are never completed.

If you are running `redis-cli monitor` in another terminal while making the requests, you will get something like the following:

```s
$ redis-cli monitor
OK
1646223690.693283 [1 [::1]:52269] "get" "rack::attack:5487412:req/ip:::1"
1646223690.693643 [1 [::1]:52269] "incrby" "rack::attack:5487412:req/ip:::1" "1"
1646223690.693819 [1 [::1]:52269] "ttl" "rack::attack:5487412:req/ip:::1"
1646223690.695247 [1 [::1]:52269] "get" "rack::attack:164622369:example:::1"
1646223690.695574 [1 [::1]:52269] "set" "rack::attack:164622369:example:::1" "1" "PX" "11000"
1646223690.716442 [1 [::1]:52269] "get" "rack::attack:5487412:req/ip:::1"
1646223690.716926 [1 [::1]:52269] "incrby" "rack::attack:5487412:req/ip:::1" "1"
1646223690.718753 [1 [::1]:52269] "ttl" "rack::attack:5487412:req/ip:::1"
1646223690.719312 [1 [::1]:52269] "get" "rack::attack:164622369:example:::1"
1646223690.720874 [1 [::1]:52269] "incrby" "rack::attack:164622369:example:::1" "1"
1646223690.721094 [1 [::1]:52269] "ttl" "rack::attack:164622369:example:::1"
1646223690.735547 [1 [::1]:52269] "get" "rack::attack:5487412:req/ip:::1"
1646223690.736037 [1 [::1]:52269] "incrby" "rack::attack:5487412:req/ip:::1" "1"
1646223690.736902 [1 [::1]:52269] "ttl" "rack::attack:5487412:req/ip:::1"
1646223690.738178 [1 [::1]:52269] "get" "rack::attack:164622369:example:::1"
1646223690.738526 [1 [::1]:52269] "incrby" "rack::attack:164622369:example:::1" "1"
1646223690.739195 [1 [::1]:52269] "ttl" "rack::attack:164622369:example:::1"
1646223690.753150 [1 [::1]:52269] "get" "rack::attack:5487412:req/ip:::1"
1646223690.753520 [1 [::1]:52269] "incrby" "rack::attack:5487412:req/ip:::1" "1"
1646223690.753694 [1 [::1]:52269] "ttl" "rack::attack:5487412:req/ip:::1"
1646223690.754465 [1 [::1]:52269] "get" "rack::attack:164622369:example:::1"
1646223690.756813 [1 [::1]:52269] "incrby" "rack::attack:164622369:example:::1" "1"
1646223690.757036 [1 [::1]:52269] "ttl" "rack::attack:164622369:example:::1"
1646223690.770427 [1 [::1]:52269] "get" "rack::attack:5487412:req/ip:::1"
1646223690.770855 [1 [::1]:52269] "incrby" "rack::attack:5487412:req/ip:::1" "1"
1646223690.771055 [1 [::1]:52269] "ttl" "rack::attack:5487412:req/ip:::1"
1646223690.771643 [1 [::1]:52269] "get" "rack::attack:164622369:example:::1"
1646223690.774295 [1 [::1]:52269] "incrby" "rack::attack:164622369:example:::1" "1"
1646223690.774718 [1 [::1]:52269] "ttl" "rack::attack:164622369:example:::1"
1646223690.786742 [1 [::1]:52269] "get" "rack::attack:5487412:req/ip:::1"
1646223690.787057 [1 [::1]:52269] "incrby" "rack::attack:5487412:req/ip:::1" "1"
1646223690.787279 [1 [::1]:52269] "ttl" "rack::attack:5487412:req/ip:::1"
1646223690.789307 [1 [::1]:52269] "get" "rack::attack:164622369:example:::1"
1646223690.789628 [1 [::1]:52269] "incrby" "rack::attack:164622369:example:::1" "1"
1646223690.789802 [1 [::1]:52269] "ttl" "rack::attack:164622369:example:::1"
1646223690.800288 [1 [::1]:52269] "get" "rack::attack:5487412:req/ip:::1"
1646223690.800639 [1 [::1]:52269] "incrby" "rack::attack:5487412:req/ip:::1" "1"
1646223690.802479 [1 [::1]:52269] "ttl" "rack::attack:5487412:req/ip:::1"
1646223690.803376 [1 [::1]:52269] "get" "rack::attack:164622369:example:::1"
1646223690.804839 [1 [::1]:52269] "incrby" "rack::attack:164622369:example:::1" "1"
1646223690.805136 [1 [::1]:52269] "ttl" "rack::attack:164622369:example:::1"
1646223690.817012 [1 [::1]:52269] "get" "rack::attack:5487412:req/ip:::1"
1646223690.817352 [1 [::1]:52269] "incrby" "rack::attack:5487412:req/ip:::1" "1"
1646223690.818606 [1 [::1]:52269] "ttl" "rack::attack:5487412:req/ip:::1"
1646223690.819539 [1 [::1]:52269] "get" "rack::attack:164622369:example:::1"
1646223690.819859 [1 [::1]:52269] "incrby" "rack::attack:164622369:example:::1" "1"
1646223690.821011 [1 [::1]:52269] "ttl" "rack::attack:164622369:example:::1"
1646223690.831110 [1 [::1]:52269] "get" "rack::attack:5487412:req/ip:::1"
1646223690.831405 [1 [::1]:52269] "incrby" "rack::attack:5487412:req/ip:::1" "1"
1646223690.833017 [1 [::1]:52269] "ttl" "rack::attack:5487412:req/ip:::1"
1646223690.833495 [1 [::1]:52269] "get" "rack::attack:164622369:example:::1"
1646223690.835958 [1 [::1]:52269] "incrby" "rack::attack:164622369:example:::1" "1"
1646223690.836259 [1 [::1]:52269] "ttl" "rack::attack:164622369:example:::1"
1646223690.848239 [1 [::1]:52269] "get" "rack::attack:5487412:req/ip:::1"
1646223690.848561 [1 [::1]:52269] "incrby" "rack::attack:5487412:req/ip:::1" "1"
1646223690.849667 [1 [::1]:52269] "ttl" "rack::attack:5487412:req/ip:::1"
1646223690.850093 [1 [::1]:52269] "get" "rack::attack:164622369:example:::1"
1646223690.851061 [1 [::1]:52269] "incrby" "rack::attack:164622369:example:::1" "1"
1646223690.851312 [1 [::1]:52269] "ttl" "rack::attack:164622369:example:::1"
```

If we use HTTPie then our successful and failed calls will look like the following:

```s
# Successful attempt
$ http GET localhost:3000/hello
HTTP/1.1 200 OK
Cache-Control: max-age=0, private, must-revalidate
Content-Type: application/json; charset=utf-8
ETag: W/"a5759911f3c834348667ca417f6c8bb4"
Referrer-Policy: strict-origin-when-cross-origin
Server-Timing: cache_read.active_support;dur=0.422119140625, cache_increment.active_support;dur=0.55322265625, start_processing.action_controller;dur=0.14404296875, process_action.action_controller;dur=0.735107421875
Transfer-Encoding: chunked
Vary: Accept
X-Content-Type-Options: nosniff
X-Download-Options: noopen
X-Frame-Options: SAMEORIGIN
X-Permitted-Cross-Domain-Policies: none
X-Request-Id: 496c2f69-8039-4715-add3-bbe57948ceb0
X-Runtime: 0.008426
X-XSS-Protection: 0

{
    "message": "Hello World"
}

# Failed attempt
$ http GET localhost:3000/hello
HTTP/1.1 429 Too Many Requests
Cache-Control: no-cache
Content-Type: text/plain
Server-Timing: cache_read.active_support;dur=7.321044921875, cache_increment.active_support;dur=0.8203125, throttle.rack_attack;dur=0.002685546875, rack.attack;dur=0.001953125
Transfer-Encoding: chunked
X-Request-Id: d8db490d-6591-405e-b70a-e9e425c35c47
X-Runtime: 0.017605

Retry later
```
