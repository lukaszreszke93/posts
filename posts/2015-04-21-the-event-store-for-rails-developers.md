---
title: "The Event Store for Rails developers"
created_at: 2015-04-21 18:15:55 +0200
kind: article
publish: false
author: Tomasz Rybczyński
tags: [ 'event', 'eventstore' ]
newsletter: :arkency_form
img: "/assets/images/events/city-people-fit.jpg"
---

<p>
  <figure align="center">
    <img src="/assets/images/events/city-people-fit.jpg">
  </figure>
</p>

We have experimented for some time with an Event Sourcing in our projects.
This is why we released free HTTP [connector](https://github.com/arkency/http_eventstore) to the Greg’s Event Store written in Ruby.
On the basis of an earlier experiences from the one of our projects we decided to create own implementation of an Event Store.
I would like to announce the first release of [Rails Event Store](https://rubygems.org/gems/rails_event_store) gem.

<!-- more -->

## Usage

If you already have `rails_event_store` gem in your Gemfile then you have to create table in your database. To do this you have to run the provided task. This will generate an activerecord migration.

```
#!ruby
rails generate rails_event_store:migrate
rake db:migrate
```

To use our gem’s functionality you have to create instance of `RailsEventStore::Client` class. 
```
#!ruby
 client = RailsEventStore::Client.new
```

#### Creating events

Creating events is very simple. At the beginning you have to define own event model extending `RailsEventStore::Event` class.

```
class ProductAdded < RailsEventStore::Event
end
 ```

Now you are prepared to create event's instance and save to a database.

```
 stream_name = "product_1"
event_data = {data: { name: "Test Product" }} 
event = ProductAdded.new(event_data) 
client.publish_event(event, stream_name) 
client.publish_event(event)
``` 

We use the concept of streams like in Greg’s Event Store but (as you can see in above example) you are able to create global event. The `event_id` is also optional value. If you leave it blank then application generate UUID for you.
The `rails_event_store` provide also optimistic concurrency control. You can define expected version of stream during creating event. In this case the last event identifier.

```
#!ruby stream_name = "product_1"
event_data = {
    data: { name: „Test Product” },
    event_id: "b2d506fd-409d-4ec7-b02f-c6d2295c7edd"
}
event = ProductAdded.new(event_data)
expected_version = "850c347f-423a-4158-a5ce-b885396c5b73"
client.publish_event(event, stream_name, expected_version)
```

#### Reading events

You can fetch events from database in a several ways. In any case, loaded events are sorted ascending.

* Reading event’s batch:
 ```
#!ruby
stream_name = "product_1"
start_event = "b2d506fd-409d-4ec7-b02f-c6d2295c7edd"
count = 40 client.read_all_events(stream_name, start_event, count)
``` 

* Reading all stream’s events:

```
#!ruby stream_name = "product_1"
client.read_all_events(stream_name)
```
 
* Reading all events:

```
#!ruby client.read_all_streams
```

#### Deleting stream

You can permanently delete all events from specific stream. Use this wisely.

```
#!ruby
stream_name = "product_1"
client.delete_stream(stream_name)
```

#### Subscription mechanism

Using our library you can synchronously listen on specific events. The only requirement is that subscriber class has to implement the 'handle_event(event)' method. Check out following example.

```
#!ruby
class InvoiceReadModel
    def handle_event(event)
        if event.event_type == 'ProductAdded’
		    create_new_product(event.data)
		end
		if event.event_type == 'ProductUpdated’
			update_product(event.data)
		end
    end
	private
	def create_new_product(event_data)
	    #Implementation here
	end
	def update_product(event_data)
	    #Implementation here
	end 
end
 
invoice = InvoiceReadModel.new
client.subscribe(invoice, ['ProductUpdated, 'ProductAdded']) 
```