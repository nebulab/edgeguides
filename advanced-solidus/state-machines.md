# State machines

## What’s a state machine?

As a programmer, state is one of the most challenging parts to model in a system. For instance, if you ask a semaphore which light is turned on, it might answer red, green, or amber, depending on when you ask it and what happened before the request: the _state_ of the semaphore depends on user-initiated and external events.

A [finite state machine](https://en.wikipedia.org/wiki/Finite-state\_machine) (FSM) is a design pattern where a system, or one of its components, can be in one and only one of a limited number of states at a given time. Transitions are allowed between states under well-known events. As long as the initial state and the list of past events are known, you can recreate a state machine at any point.

For instance, a semaphore can be in a red, green, or amber state. When a pedestrian presses a button, it transitions from green to amber. When on amber, it goes to red if 5 seconds pass. If red, it turns green after 180 seconds. If you know a semaphore was green at time zero, and the list of events has been button-press, 500 seconds, button-press, 3 seconds, you know it's amber now.

Several parts of an e-commerce system fit this paradigm well. An order can be in progress, waiting for payment, or completed. It goes to completed when the payment is made. At that point, reimbursement can still be processed o completed. A payment... well, you get the idea.

## State machines in Solidus

Solidus defines [a few state machines](https://github.com/solidusio/solidus/tree/master/core/lib/spree/core/state\_machines) on top of some models.

Each state machine describes its valid states and the allowed transitions. It also defines event methods that can be called from the outside to trigger internal changes. Finally, they can also declare some hooks that run when specific transitions happen.&#x20;

{% hint style="info" %}
Internally, Solidus' state machines use the [`states_machine`](https://github.com/state-machines/state\_machines) gem (more precisely, [`states_machine-activerecord`](https://github.com/state-machines/state\_machines-activerecord)). Take a look at [`states_machine`'s README](https://github.com/state-machines/state\_machines/blob/master/README.md) for more details on its usage and API.
{% endhint %}

## Customizing states machines

State machines modules are included in the corresponding model, so all the strategies described in the [core customization section](../customization/customizing-the-core.md) are valid.

{% hint style="danger" %}
It's better to be conservative when customizing state machines. Try to apply the smallest possible set of changes, and if possible, avoid changing the defined states. You're dealing with the core of the domain model: large changes could branch out in unanticipated ways!
{% endhint %}

### Customizing core behavior

{% hint style="danger" %}
Be aware of not overusing transition hooks in the state machines. When the involved logic requires reaching external services or, more generally, is decoupled from the main flow, you're better off [leveraging the event bus](state-machines.md#adding-orthogonal-behavior). Otherwise, you will eventually run into the typical gotchas and downsides of abusing `ActiveRecord` callbacks.
{% endhint %}

Sometimes you might need to tweak Solidus' core model to fit your business needs. In that case, you might want to tweak a state machine to obey your extended domain.

Say that you must store the time when a payment has been marked as completed. You already added a `completed_at` column to the `spree_payments` table, but now you need the payment state machine to fill it.

You can add an `after_transition hook` using an [override](../customization/customizing-the-core.md#using-overrides):

{% code title="app/overrides/my_store/payment_set_completed_at.rb" %}
```ruby
# frozen_string_literal: true

module MyStore
  module PaymentSetCompletedAt
    def self.prepended(base)
      base.state_machine.after_transition(to: :completed) do
        self.completed_at = Time.zone.now
      end
    end

    ::Spree::Payment.prepend self
  end
end
```
{% endcode %}

### Adding orthogonal behavior

{% hint style="warning" %}
Don't be confused about state machine events vs. bus events. State machine events are conditions that can produce a transition between valid states. They're local to the state machine component. On the other hand, bus events can be published and consumed anywhere within the system and, per se, have nothing to do with the state machines.
{% endhint %}

As noted in the [event bus](../customization/subscribing-to-events.md) guide, you can leverage the event bus to hook into core events. That's helpful when you need to perform something in response to a change in the system, but your logic is orthogonal (i.e., decoupled) to the main flow. Transitions between state machine states are good candidates to become hotspots where tangential logic is triggered.

For instance, you might want to update your ERP or send an SMS when a payment is marked as completed. The cleaner way to do that is to publish an event when that happens and then subscribe to it. First, you need to override the `#complete` event method on `Spree::Payment` (see the [overrides section](../customization/customizing-the-core.md#using-overrides) for the required setup code):

{% code title="app/overrides/my_store/publish_payment_completed.rb" %}
```ruby
# frozen_string_literal: true

module MyStore
  module PublishPaymentCompleted
    def complete
      super.tap do |result|
        Spree::Bus.publish(:payment_completed, payment: self) if result
      end
    end

    ::Spree::Payment.prepend self
  end
end
```
{% endcode %}

Then, after [registering the new event](../customization/subscribing-to-events.md#custom-events), you can [create a subscriber](../customization/subscribing-to-events.md#subscription-to-events) for it.

{% hint style="warning" %}
Why not register a new `after_transition` hook instead of overriding the event method (i..e, `#complete` in the example above)?

State machine transitions (including their `after_` hooks) are wrapped within a database transaction. By overriding the method instead of using `after_transition`, we publish the bus event only after the transaction has been committed to the DB.

Event subscribers should **always** be decoupled from the main transaction, so there's no point in blocking database access until your subscribers have finished running. In fact, it's a bad practice: a failed subscriber will roll back the whole DB transaction and potentially leave your system in an inconsistent state.
{% endhint %}

{% hint style="info" %}
Ideally, Solidus would publish events for every state machine transition out of the box. Our event bus is fairly new and we're still working on it, but we'll get there eventually! In the meantime, you can check`Spree::Bus.registered_events` for the complete list of events that are already published.
{% endhint %}

### Using custom state machines

If needed, you can flat out replace state machines with your custom implementation. This can be done through the `state_machines` option in `config/initializers/spree.rb`.

For instance, if you want to replace the payment state machine, you can do it like this:

{% code title="lib/my_store/state_machines/payment.rb" %}
```ruby
# frozen_string_literal: true

module MyStore
  class StateMachines
    module Payment
      extend ActiveSupport::Concern

      included do
        state_machine initial: :custom_state do
          # Event, transition & hook definitions
        end
      end
    end
  end
end
```
{% endcode %}

And then you need to tell Solidus to use it:

{% code title="config/initializers/spree.rb" %}
```ruby
# ...
Spree.config do |config|
  config.state_machines.payment = 'MyStore::StateMachines::Payment'
  # ...
end
```
{% endcode %}

{% hint style="warning" %}
As always, with great power comes great responsibility. Replacing the whole state machine should be the outcome of an informed decision. Solidus relies on well-known state machine states and events in many areas of the core, so be prepared to adjust other parts of Solidus to work with your custom implementation.
{% endhint %}
