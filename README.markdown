Couchbase Ruby Client
=====================

This is the official client library for use with Couchbase Server.

INSTALL
=======

This gem depends on a couple of external libraries to work with JSON and
HTTP. For JSON it uses [yajl-ruby][1] and [yaji][2] which are built atop
of [yajl][3]. For HTTP iteraction it uses [curb][4], the curl bindings for
ruby. To install it use your package manager. On debian you can install
it, use `aptitude`:

    $ sudo aptitude install libcurl3-dev

After that you can install the gem via rubygems:

    $ gem install couchbase

USAGE
=====

First of all you need to load library:

    require 'couchbase'

To establish connection with couchbase you need to specify at least pool
URI. Also you can choose custom bucket name and memcached SASL
authentication and Couchbase REST API.

    c = Couchbase.new("http://localhost:8091/pools/default")
    c = Couchbase.new("http://localhost:8091/pools/default", :bucket_name => 'blog')
    c = Couchbase.new("http://localhost:8091/pools/default",
                      :bucket_name => 'blog', :bucket_password => 'secret')

This gem supports memcached API accessible for [memcached][5] client
with a bit syntax sugar:

    c.set('password', 'secret')
    c.get('password')             #=> "secret"
    c['password'] = 'secret'
    c['password']                 #=> "secret"
    c['counter'] = 1              #=> 1
    c['counter'] += 1             #=> 2
    c['counter']                  #=> 2
    c.increment('counter', 10)
    c.flush

But if you store structured data, they will be treated as documents and
you can handle them in map/reduce function from CouchDB views. For
example, store a couple of posts using memcached API:

    c['biking'] = {:title => 'Biking',
                   :body => 'I went to the the pet store earlier and brought home a little kitty...',
                   :date => '2009/01/30 18:04:11'}
    c['bought-a-cat'] = {:title => 'Biking',
                         :body => 'My biggest hobby is mountainbiking. The other day...',
                         :date => '2009/01/30 18:04:11'}
    c['hello-world'] = {:title => 'Hello World',
                        :body => 'Well hello and welcome to my new blog...',
                        :date => '2009/01/15 15:52:20'}
    c.all_docs.count    #=> 3

Now let's create design doc with sample view and save it in file
'blog.json':

    {
      "_id": "_design/blog",
      "language": "javascript",
      "views": {
        "recent_posts": {
          "map": "function(doc){if(doc.date && doc.title){emit(doc.date, doc.title);}}"
        }
      }
    }

This design document could be loaded into the database like this (also you can
pass the ruby Hash or String with JSON encoded document):

    c.save_design_doc(File.open('blog.json'))

To execute view you need to fetch it from design document `_design/blog`:

    blog = c.design_docs['blog']
    blog.views                    #=> ["recent_posts"]
    blog.recent_posts             #=> [#<Couchbase::Document:14244860 {"id"=>"hello-world", "key"=>"2009/01/15 15:52:20", "value"=>"Hello World"}>,...]

Gem uses streaming parser to access view results so you can iterate them
easily and if your code won't keep links to the documents GC might free
them as soon as it decide they are unreachable, because parser doesn't
store global JSON tree.

    posts = blog.recent_posts.each do |doc|
      # do something
      # with doc object
    end

You can also use Enumerator to iterate view results

    require 'date'
    posts_by_date = Hash.new{|h,k| h[k] = []}
    enum = c.all_docs.each  # request hasn't issued yet
    enum.inject(posts_by_date) do |acc, doc|
      acc[date] = Date.strptime(doc['date'], '%Y/%m/%d')
      acc
    end

The Couchbase server could generate errors during view execution with
`200 OK` and partial results. By default the library raises exception as
soon as errors detected in the result stream, but you can define the
callback `on_error` to intercept these errors and do something more
useful.

    view = blog.recent_posts
    logger = Logger.new(STDOUT)

    view.on_error do |from, reason|
      logger.warn("#{view.inspect} received the error '#{reason}' from #{from}")
    end

    posts = view.each do |doc|
      # do something
      # with doc object
    end

Note that errors object in view results usually goes *after* the rows,
so you will likely receive a number of view results successfully before
the error is detected.

[1]: https://github.com/brianmario/yajl-ruby/
[2]: https://github.com/avsej/yaji/
[3]: http://lloyd.github.com/yajl/
[4]: https://rubygems.org/gems/curb/
[5]: https://github.com/fauna/memcached/