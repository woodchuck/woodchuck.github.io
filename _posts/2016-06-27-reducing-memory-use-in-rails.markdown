---
layout: post
title:  "Reducing memory use in Rails"
date:   2016-06-27
categories: rails performance memory
---

Often, in rake tasks we iterate over large numbers of records. This can easily lead to allocating all available memory on the system if we're not careful. Below are some ways to dramatically lower memory use in these cases.

Anytime we assign to a variable, memory has to be allocated for that. Ruby won't be able to garbage collect that memory until the variable goes out of scope and nothing references what it pointed to anymore. Variables containing rails ActiveRecord object instances will take up much more memory than simple strings if they contain a lot of attributes. So we want to avoid instantiating a lot of AR objects at once and keeping references to them around.

### Updating Records Based On Combined Scope Criteria

#### High Memory

{% highlight ruby %}
users_with_innie_belly_button = User.with_innie_belly_button
users_who_are_dead = User.deceased
users_who_are_eligible_for_nougies = User.eligible_for_nougies
users_to_update = users_who_are_eligible_for_nougies \
                    - users_with_innie_belly_button \
                    - users_who_are_dead
users_to_update.each do |u|
  u.update_attributes(foo: "New improved foo")
end
{% endhighlight %}

This will instantiate all of the Users in the 3 scopes, whether or not we ultimately update them or not. Also, if we make multiple updates based on different combinations of criteria, ruby won't be able to reclaim the memory held in one of these collections until after the last reference to the variable that contains it.

#### Low Memory

{% highlight ruby %}
User.
  eligible_for_nougies.
  living.
  with_outie_belly_button.
  update_all(foo: "New improved foo")
{% endhighlight %}

This doesn't instantiate any Users. It runs a single SQL update against the DB directly, and is far faster. It's also more clear to read.

One challenge is that rails 4 doesn't support `.or`. In rails 5 you can do:

{% highlight ruby %}
User.
  with_outie_belly_button.
  or(User.elligible_for_nougies).
  living.
  update_all(foo: "New improved foo")
{% endhighlight %}

You may also want to negate an existing scope that involves some complex logic, rather than creating an additional scope. For instance .eligible_for_nougies vs. .inelligible_for_nougies. In rails 4.2 you can do:

{% highlight ruby %}
User.
  living.
  where.not(id: User.elligible_for_nougies)
{% endhighlight %}

### Iterating Over Records

#### High Memory

{% highlight ruby %}
User.all.each do |user|
  # Do something with user
end
{% endhighlight %}

This is the natural way to iterate all records for a model, but if we have 10,000 Users, this will instantiate objects for all of them at once.

#### Low Memory

{% highlight ruby %}
User.find_each(batch_size: 100) do |user|
  # Do something with user
end
{% endhighlight %}

This will only instantiate 100 Users at a time. That's 1% of the memory `User.all.each` consumes if there are 10,000 Users. `batch_size` defaults to 1,000.

### Eager Loading

#### Higher Memory

{% highlight ruby %}
scope :elligible_for_nougies, -> {
  includes(:subscriptions).
  where(subscriptions: { name: "Enterprise Nougies" }
}
{% endhighlight %}

This eager loads the associated Subscription objects and includes them in the instantiated User objects.

#### Lower Memory

{% highlight ruby %}
scope :elligible_for_nougies, -> {
  joins(:subscriptions).
  where(subscriptions: { name: "Enterprise Nougies" }
{% endhighlight %}

This only uses the join to identify which Users to return, and won't instantiate any Subscription objects by default. However, if you access the .subscriptions association of a User instance it will perform an N+1 query.
