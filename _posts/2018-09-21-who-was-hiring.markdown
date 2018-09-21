---
layout: post
title:  "Ask HN: Who was hiring?"
date:   2018-09-21
categories:
---

This post is about techniques you can use to collect old "Who is hiring?" content
from Hacker News.

![who is hiring thread]({{ "/assets/who_is_hiring.png" | absolute_url }})

## What is Hacker News?

[Hacker News](https://news.ycombinator.com/) is a Reddit style website run by US based startup incubator [Y Combinator](http://www.ycombinator.com/).
Users submit content (usually tech related news) and other users vote the content up or down. 

My favourite recurring story is the monthly ["Who is hiring?"](https://www.google.com/search?q=site%3Anews.ycombinator.com+%22Who+is+hiring%3F%22) 
where tech
companies from around the world post links to current vacancies. These
ads are interesting since they are posted by internal recruiters and
are usually transparent in terms of who the company is and what they do.

## Why look at old stories?

If you are looking for a job it makes sense to look at the most recent posts, 
but I find old posts interesting as well since:

1. Those companies might still be hiring
2. The posts give some insight (through the HN lens) as to what the 
   job market is like in a particular city
3. I can see what kind of technologies companies are using

## Getting links to stories

Before you can download historical "Who is hiring?" content you need to 
find links to the stories. This Ruby script does just that using the Google
search API:

{% highlight ruby %}
require 'rest-client'
require 'json'

results = []
index = 1

begin

  loop do
  
    obj = JSON.parse(
      RestClient.get("https://www.googleapis.com/customsearch/v1", {
        :params => {
          key: "<redacted>", 
          cx: "<redacted>", 
          dateRestrict: "y5", 
          start: index,
          q: "\"Ask HN: Who is Hiring\""
        } 
      }).body.to_s
    )
    
    obj['items'].each do |item|
    
      if /Ask HN: Who is hiring\? \([a-zA-Z]+ [0-9]+\)/.match(item['title'])
    
        results << item.select{|k,v|['link','title'].include? k}
        
      end
      
    end
    
    index = obj['queries']['nextPage'].first['startIndex']
    
  end
  
rescue => e
  STDERR.puts e
end

puts JSON.pretty_generate(results)
{% endhighlight %}

I'm searching for the exact phrase "Ask HN: Who is hiring?" using 
a custom search engine configured to only search HN. The search
results are limited to the last five years. I'm also using some regex
to make sure I don't collect links for meta stories.

Google returns 10 results per page and (at time of writing) will never return more than 10 pages 
regardless of how many results there are. The script will loop until an exception is encountered;
for me this is a non-successful response code to the request for the 11th page of results.

This is how I run the script:

{% highlight terminal %}
$ ruby search.rb > links.json
{% endhighlight %}

## Getting the content

Once you have the links, you can use the [HN API](https://github.com/HackerNews/API) to get the content. This is my script:

{% highlight ruby %}
require 'json'
require 'restclient'
require 'logger'

logger = Logger.new(STDERR)

RQ = Struct.new(:id, :output)

input = Queue.new
output = Queue.new

pool = Array.new(60) do

  Thread.new do
  
    loop do
  
      rq = input.pop
      
      break unless rq      
      
      begin
      
        item = JSON.parse(
          RestClient.get("https://hacker-news.firebaseio.com/v0/item/#{rq.id}.json").body.to_s
        )
      
        if item
          rq.output.push(item)
        else
          logger.info "#{rq.id} returned a NULL object"
        end

      rescue => e
        logger.info "#{rq.id} failed but will try again (#{e})"
        input.push(rq)
      end
       
    end
    
  end
  
end

writer = Thread.new do

  loop do
  
    post = output.pop
  
    if post.nil?
      logger.info "finished writing"    
      break
    end
  
    puts post.to_json
    
  end

end

JSON.parse(File.read(ARGV[0])).each do |page|
  
  rq = RQ.new(/id=(?<item>[0-9]+)/.match(page['link'])[:item], Queue.new)
  
  logger.info "getting '#{page['title']}'..."
  
  input.push(rq)
  
  post = rq.output.pop
  
  logger.info "getting #{post['kids'].size} posts"
  
  post['kids'].each do |id|
    
    input.push(RQ.new(id, output))
  
  end
  
end

pool.each{input.push(nil)}.each(&:join)
[writer].each{output.push(nil)}.each(&:join)

logger.info "all done"
{% endhighlight %}

The script gets each story object and then uses that information to get all 
of the top level posts.

The HN API seems to take around 1s to answer a request and doesn't return collections.
This means if a story contains 1000 top level posts, it will take ~16 minutes to get them all if you
only use one thread of execution. To speed this up my script uses a pool of worker threads. I assume 
this isn't a burden for the host since it's Firebase and every app using this API will need
to do a lot of IO.

The script must not be left running unattended since failed requests are re-queued regardless of the type 
of failure. This is a lazy solution to recovering from the occasional socket related temporary failure.

Running the script produces STDERR terminal output that looks like this:

{% highlight terminal %}
$ ruby get.rb links.json > posts.json
I, [2018-09-03T12:20:48.334370 #859]  INFO -- : getting 'Ask HN: Who is hiring? (August 2018) | Hacker News'...
I, [2018-09-03T12:20:49.000806 #859]  INFO -- : getting 918 posts
I, [2018-09-03T12:20:49.025621 #859]  INFO -- : getting 'Ask HN: Who is hiring? (July 2018) | Hacker News'...
I, [2018-09-03T12:20:53.952558 #859]  INFO -- : 17664917 returned a NULL object
I, [2018-09-03T12:21:01.930408 #859]  INFO -- : getting 776 posts
I, [2018-09-03T12:21:01.932161 #859]  INFO -- : getting 'Ask HN: Who is hiring? (June 2018) | Hacker News'...
I, [2018-09-03T12:21:08.996272 #859]  INFO -- : 17493081 returned a NULL object
I, [2018-09-03T12:21:12.678740 #859]  INFO -- : getting 796 posts
I, [2018-09-03T12:21:12.690395 #859]  INFO -- : getting 'Ask HN: Who is hiring? (May 2018) | Hacker News'...
I, [2018-09-03T12:21:23.509626 #859]  INFO -- : getting 1033 posts
I, [2018-09-03T12:21:23.514804 #859]  INFO -- : getting 'Ask HN: Who is hiring? (March 2018) | Hacker News'...
I, [2018-09-03T12:21:30.793308 #859]  INFO -- : 16987633 returned a NULL object
I, [2018-09-03T12:21:34.502299 #859]  INFO -- : 16969165 returned a NULL object
I, [2018-09-03T12:21:36.262272 #859]  INFO -- : 16970062 failed but will try again (SSL_connect SYSCALL returned=5 errno=0 state=unknown state)
I, [2018-09-03T12:21:37.664146 #859]  INFO -- : getting 909 posts
...
{% endhighlight %}

It took me around 10 minutes to return ~40,000 top level posts from the past 5 years.

## Filter by keyword

Once you have all the posts from the previous step you can process them any way you like. 
I decided to filter the results by keywords and create human readable lists.

This is my keyword filter:

{% highlight ruby %}
require 'json'

def self.filter(file, **opts)

  results = []
  
  f = {
    
    :all => ( opts[:all] ? opts[:all].each(&:downcase).uniq.sort : [] ),
    :any => ( opts[:any] ? opts[:any].each(&:downcase).uniq.sort : [] ),
    :none => ( opts[:none] ? opts[:none].each(&:downcase).uniq.sort : [] ),
  }

  pattern = Regexp.union((f[:all] + f[:any] + f[:none]).map{|s|Regexp.new("\\b" + Regexp.escape(s) + "\\b", Regexp::IGNORECASE)})

  File.open(file, "r").each do |line|

    if post = JSON.parse(line)
      
      if post['text']
      
        matches = post['text'].scan(pattern).map(&:downcase).uniq.sort
        
        if f[:all].empty? or ((matches & f[:all]).sort == f[:all])
        
          if f[:any].empty? or (matches & f[:any]).size > 0
          
            if (matches & f[:none]).empty?
            
              results << post
              
            end
            
          end
        
        end
          
      end
    
    end
    
  end

  results

end
{% endhighlight %}

The script scans for a union of keywords in the post and returns the results as an array. 
I use this array to accept/reject the post based on:

- keywords which must all be present (all)
- keywords where one or more must present (any)
- keywords which must not be present (none)

I print the filtered posts in reverse-chronological order with HTML formatting converted to plain-text using this script:

{% highlight ruby %}
require 'time'
require 'html_to_plain_text'

def self.put_list(f, results)
  
  f.puts "#{results.size} results"
  f.puts ""

  results.sort_by{|post|post['time']}.reverse.each do |post|
    
    f.puts Array.new(80){"-"}.join
    f.puts "#{post['by']} "
    f.puts "https://news.ycombinator.com/user?id=#{post['by']}"
    f.puts "#{Time.at(post['time'])}"
    f.puts ""
    f.puts HtmlToPlainText.plain_text(post['text'])
    f.puts ""
    
  end
  
end
{% endhighlight %}

The filter and print scripts are implemented as methods so they can be
parametrised with different keywords like this:

{% highlight ruby %}
File.open("london_c.txt", "w") do |f|
  put_list(
    f,
    filter(ARGV[0], 
      all: %w(london),
      any: %w(c c++ c/c++ firmware embedded)
    )
  )
end
{% endhighlight %}

{% highlight ruby %}
File.open("berlin_ruby.txt", "w") do |f|
  put_list(
    f,
    filter(ARGV[0], 
      all: %w(berlin ruby)
    )
  )
end
{% endhighlight %}

And so on.

## Finally

If you are actively looking for work on HN I recommend [this](https://hnhired.com/) app
since it more or less does what my scripts do but in your browser and for the most recent month.
