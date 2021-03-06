---
title: "Using domain events as success/failure messages"
created_at: 2015-05-15 12:59:51 +0200
publish: true
author: Mirosław Pragłowski
tags: [ 'rails_event_store', 'domain event', 'event sourcing' ]
newsletter: arkency_form
img: "events/failure_domain_event.jpg"
---

<p>
  <figure>
    <img src="<%= src_fit("events/failure_domain_event.jpg") %>" width="100%">
  </figure>
</p>

## When you publish an event on success make sure you publish on failure too

We had an issue recently with one of our internal gems used to handle all communication with a external payment gateway. We are using gems to abstract a bounded context (payments here) and to have an abstract anti-corruption layer on top of external system's API.

<!-- more -->

When our code is triggered (no matter how in scope of this blog post) we are using our gem's methods to handle payments.

```ruby
# ...
payment_gateway.refund(transaction)
# ...
```

There are different payment gateways - some of them respond synchronously some of them prefer asynchronous communication. To avoid coupling we publish an event when the payment gateway responds.

```ruby
class TransactionRefunded < RailsEventStore::Event

class PaymentGateway
  RefundFailed = Class.new(StandardError)

  def initialize(event_store = RailsEventStore::Client.new, api = SomePaymentGateway::Client.new)
    @api = api
    @event_store = event_store
  end

  def refund(transaction)
    api.refund(transaction.id, transaction.order_id, transaction.amount)
    transaction_refunded(transaction)
  rescue
    raise RefundFailed
  end
  # there are more but let's focus only on refunds now

  private
  attr_accessor :event_store, :api

  def transaction_refunded(transaction)
    event = TransactionRefunded.new({ data: {
      transaction_id: transaction.id,
      order_id:transaction.order_id,
      amount:transaction.amount }})
    event_store.publish(event, order_stream(transaction.order_id)
  end

  def order_stream(order_id)
    "order$#{order_id}"
  end
end
```
(a very simplified version - payments are much more complex)


You might have noticed that when our API call fails we rescue an error and raise our one. It is a way to avoid errors from the 3rd party client leak to our application code. Usually that's enough and our domain code will cope well with failures.

But recently we got a problem. The business requirements are: _When refunding a batch of transactions gather all the errors and send them by email to support team to handle them manually_.

That we have succeeded to implement correctly. One day we have received a request to explain why there were no refunds for  a few transactions.

## And then it was trouble

The first thing we did was to check history of events for the aggregate performing the action (Order in this case). We have found an entry that refund of order was requested (it is done asynchronously) but there were no records of any transaction refunds.

It could not be any. Because we did not publish them :( This is how this code should look like:

```ruby
class TransactionRefunded < RailsEventStore::Event
class TransactionRefundFailed < RailsEventStore::Event

class PaymentGateway
  #... not changed code omitted

  def refund(transaction)
    # ...
    transaction_refunded(transaction)
  rescue => error
    transaction_refund_failed(transaction, error)
  end

  def transaction_refunded(transaction)
    publish(TransactionRefunded, transaction)
  end

  def transaction_refund_failed(transaction, error)
    publish(TransactionRefundFailed, transaction) do |data|
      data[:error] = error.message
    end
  end

  # helper method to publish both kind of events with similar data
  def publish(event_type, transaction)
    event_data = { data: {
      transaction_id: transaction.id,
      order_id:       transaction.order_id,
      amount:         transaction.amount
    }}.tap do |data|
      yield data if block_given?
    end
    event = event_type.new(data)
    event_store.publish(event, order_stream(transaction.order_id)
  end
end
```

The raise of an error here was replaced by the use of domain event. What is raising an error? It is a domain event ... when domain is the code. By publishing our own domain event we give it a business meaning. Check the Andrzej's blog post [Custom exceptions or domain events?](http://andrzejonsoftware.blogspot.com/2014/06/custom-exceptions-or-domain-events.html) for more details.

## But wait, why not just change the error handling?

Of course we could do it without the use of domain events that are persisted in [Rails Event Store](https://github.com/arkency/rails_event_store) but the possibility of going back in the history of the aggregate is priceless. Just realise that a stream of the domain events that are responsible for changing the state of an aggregate are the full audit log that is easy to present to the user.

And one more thing: you want to have a monthly report of failed refunds of transactions? Just implement a handler for TransactionRefundFailed the event and do your grouping, summing & counting and store the results. And by replaying all past TransactionRefundFailed events with use of your report building handler you will get a report for the past months too!
