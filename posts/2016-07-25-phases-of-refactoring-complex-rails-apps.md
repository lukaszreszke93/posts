---
title: "Phases of refactoring complex Rails apps"
created_at: 2016-07-25 12:59:37 +0200
publish: true
tags: [ 'rails', 'refactoring' ]
author: Marcin Grzywaczewski
---


Refactoring is a on-going process that is necessary in big Rails applications. Without it, you can quickly get into trouble - not only your code quality will suffer, but pieces of your architecture - models, controllers and views will get more and more coupled together. This is not a nice separation you had at the beginning of the project - it can quickly transform into an entangled mess of callbacks, going by relationships through half of the system to do stuff, and horrible things like that.

<!-- more -->


Sure, you can live with that. But every feature you’ll add, every bug fix you’ll make will be harder and harder.

You can fight with it. Add some gems, clean up one piece of entangled code. But those are small improvements - they can eventually sum up to a big improvement, but it’s unlikely.

Not to mention time won’t stop - there is a requirement from the business that new features will get delivered. And they aren’t aware of problems you can experience in your code. Even if they are, it is often resulting in a loss of trust. Just think about it - would you trust a car mechanic which says something like “you know, this fix of brake system will take more time because I’ve made mess in the engine after fixing it before”? I would not. I’d say it’d be very unprofessional :).

And this lose of trust will snowball and get you into trouble - more control, more meetings. Even less things done.

You can avoid all those problems by understanding simple (but brutal :() truths. 

The first truth is that the Rails Way does not scale. It works cool for CRUD apps. It works great at the beginning of the project. It also works well for simple domains. That’s why you hear that big, scaling projects are using Rails - but in fact it’s not because Rails Way is scaling. It is because they did a lot of work to make it right, or their business domain is simple enough to be fitted into the ‘CRUD’ approach.

And the second truth - you won’t get far in terms of results if you refactor your code without a plan. Small improvements are great and they’re better than doing nothing. But in fact major problems are solved by **modeling them away**. We call this way ‘the New Way’ or ‘Post-Rails’.

You don’t need to model those problems away by yourself. You have *years* of experience of software developers & architects to support you. Rails is far away from good OOP principles - and when problems happen, it’s very wise to resort to them. In Arkency we have a ‘framework’ for escaping from framework ;). There are certain techniques that are powerful and fixes the first fundamental problem of Rails frameworks. We think that is a great refactoring plan you can apply to your project _right now_ - and we want to share it with you.

## Regain control over your controllers by introducing missing architectural pieces.

Controllers are tricky in Rails. They break the single responsibility principle in a rather brutal way. They orchestrate your models to do stuff. They take care of HTTP request parameters processing. They set up shared state across your actions. They take the responsibility for orchestrating rendering of views. They choose over many response formats based on the content type & accept headers of your request.

We think it’s fine to give those HTTP responsibilities to a controller, but it’s very restricting to orchestrate your business logic within it. The first step to make your complex Rails app better is to get rid of this coupling. From variety of reasons - keeping the business logic separated will make you able to just extract supportive pieces into a gem, for example. The second reason is that controllers have the big sin of taking away the power of instantiating your own objects.

To regain control, the following pieces needs to be introduced - form & service objects.

### Form objects

Form objects are all about taking away the params processing responsibility out of the controller. In form objects you’re making proper type coercions for your attributes, as well as providing simple validations - like presence, numericality & length validations. In Arkency it’s usually implemented using [Virtus](https://github.com/solnic/virtus) & [ActiveModel](http://api.rubyonrails.org/classes/ActiveModel/Model.html) libraries, but we’re looking forward to use [dry-types]() instead of Virtus for them.

What you do with form objects is just wrapping `params` object from controller with them:

```ruby
class SubmitArticleForm
	include ActiveRecord::Validations
	include Virtus.model
	
	attribute :title, String
	attribute :content, String
	
	validates :title, :content, presence: true
	validates :title, length: { minimum: 5 }	
	
	def persisted?
		false
	end
	
	def validate!
		raise ValidationError.new(errors: errors) unless valid?
	end
end
```

And then use it in your controller:

```ruby
class ArticlesController < ApplicationController
	def create
		form = SubmitArticleForm.new(params[:article])
		form.validate!
		
		# your logic here.
	rescue ValidationError => err
		# …
	end
end
```

But having only form objects is not very helpful. Form objects are far more useful if combined with the service object pattern. You can [read more](http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/) about form objects (point #3) in this great article - but in Arkency there is no `save!` method in form objects.

### Service objects

While form objects are extracting the input handling responsibility from your controllers, service objects are all about extracting business logic out of them. They don’t need any  supportive technologies - they are just plain old Ruby objects.

```ruby
class SubmitArticle
	def initialize(article_mailer)
		@article_mailer = article_mailer
	end
	
	def call(form)
		Article.create!(form.attributes).tap do |article|
			article_mailer.published_article_mail(article).deliver_later
		end
	end
	
	private
	attr_reader :article_mailer
end
```

Then, in your controller, instead of:

```ruby
class ArticlesController < ApplicationController
	def create
		form = SubmitArticleForm.new(params[:article])
		form.validate!
		@article = Article.create!(form.attributes)
		ArticleMailer.published_article_mail(article).deliver_later
	end
end
```

You do:

```ruby
class ArticlesController < ApplicationController
	def create
		form = SubmitArticleForm.new(params[:article])
		submit_article = SubmitArticle.new(ArticleMailer)
		@article = submit_article.(form)
	end
end
```

Since `SubmitArticle` is a plain object, you can inject your dependencies - a thing which is very useful for testing. There are many things you can do further with service objects - this is the most simple implementation of such object, still relying on implicit rendering of Rails views. You can [read more](http://blog.arkency.com/2014/10/instantiating-service-objects/) about this pattern on our blog.

Those two patterns can serve you long. With even more complex patterns there are many other techniques, which will get described shortly.

### … more?

Next techniques you can use need a proper understanding of the business domain - a thing you need to learn to crunch properly. 

But this is only Arkency way, right? In fact, we’re not the only one inspired by proper design and architectural patterns. We write about them (a lot!), but there are other great developers sharing the same goal with us - make Rails a better fit for complex applications.

That's why we have organized a book bundle with those authors. Officially we've finished it on Friday, but due to many requests, we've reopened the [Post-Rails-Way Book Bundle](http://railsbookbundle.com) today, just for one day!



