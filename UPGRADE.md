# Upgrade Guide

Follow this guide to upgrade older pay versions. These may require database migrations and code changes.

## Pay 2.x to Pay 3.0

This is a major change to add support for multiple payment methods, fixing bugs, and improving the architecture of Pay.

### Database Migration

Upgrading from Pay 2.x to 3.0 requires moving data for several things:

1. Move `processor` and `processor_id` from billable models to the new Pay::Customer model
2. Associate existing `Pay::Charge` and `Pay::Subscription` records with new `Pay::Customer` records.
3. Sync default card for each `Pay::Customer` to `Pay::PaymentMethod` records (makes API requests)
4. Update `Pay::Charge` payment method details for each record to the new format
5. Convert generic trials from billable model to FakeProcessor subscriptions with trial
6. Drop unneeded columns

Here's an example migration for migrating data. This migration is purely an example for reference. Please modify this migration as needed.

```ruby
class UpgradeToPayVersion3 < ActiveRecord::Migration[6.0]
  # List of models to migrate from Pay v2 to Pay v3
  MODELS = [User, Team]

  def self.up
    # Migrate models to Pay::Customer
    MODELS.each do |klass|
      klass.where.not(processor: nil).find_each do |record|
        # Migrate to Pay::Customer
        pay_customer = Pay::Customer.where(owner: record, processor: record.processor, processor_id: record.processor_id).first_or_initialize
        pay_customer.update!(
          default: true,

          # Optional: Used for Marketplace payments
          data: {
            stripe_account: record.try(:stripe_account),
            braintree_account: record.try(:braintree_account),
          }
        )

        # Associate Pay::Charges with new Pay::Customer
        Pay::Charge.where(owner_type: record.class.name, owner_id: record.id).each do |charge|
          # Since customers can switch between payment processors, we have to find or create
          customer = Pay::Customer.where(owner: record, processor: charge.processor).first_or_create!
          charge.update!(customer: customer)
        end

        # Associate Pay::Subscription with new Pay::Customer
        Pay::Subscription.where(owner_type: record.class.name, owner_id: record.id).each do |subscription|
          # Since customers can switch between payment processors, we have to find or create
          customer = Pay::Customer.where(owner: record, processor: subscription.processor).first_or_create!
          subscription.update!(customer: customer)
        end

         # Migrate to Pay::PaymentMethod
        if record.card_type?
          # Lookup default payment method via API and create them as Pay::PaymentMethods
          begin
            case pay_customer.processor
            when "braintree"
              payment_method_id = pay_customer.customer.payment_methods.find(&:default?)
              Pay::Braintree::PaymentMethod.sync(payment_method_id) if payment_method_id
            when "stripe"
              payment_method_id = pay_customer.customer.invoice_settings.default_payment_method
              Pay::Stripe::PaymentMethod.sync(payment_method_id) if payment_method_id
            end
          rescue
          end
        end
      end

      # Migrate Pay::Charge payment method details
      Pay::Charge.find_each do |charge|
        # Data column should be a hash. If we find a string instead, replace it
        charge.data = {} if charge.data.is_a?(String)

        case charge.card_type.downcase
        when "paypal"
          charge.update(payment_method_type: :paypal, brand: "PayPal", email: charge.card_last4)
        else
          charge.update(payment_method_type: :card, brand: charge.card_type, last4: charge.card_last4, exp_month: charge.card_exp_month, exp_year: charge.card_exp_year)
        end
      end

      # Migrate generic trials
      # Anyone on a generic trial gets a fake processor subscription with the same end timestamp
      klass.where("trial_ends_at >= ?", Time.current).find_each do |record|
        # Make sure we don't have any conflicts when setting fake processor as the default
        Pay::Customer.where(owner: record, default: true).update_all(default: false)

        pay_customer = Pay::Customer.where(owner: record, processor: :fake_processor, default: true).first_or_create!
        pay_customer.subscribe(
          trial_ends_at: record.trial_ends_at,
          ends_at: record.trial_ends_at,

          # Appease the null: false on processor before we remove columns
          processor: :fake_processor
        )
      end
    end

    # Drop unneeded columns
    remove_column :pay_charges, :owner_type
    remove_column :pay_charges, :owner_id
    remove_column :pay_charges, :processor
    remove_column :pay_charges, :card_type
    remove_column :pay_charges, :card_last4
    remove_column :pay_charges, :card_exp_month
    remove_column :pay_charges, :card_exp_year
    remove_column :pay_subscriptions, :owner_type
    remove_column :pay_subscriptions, :owner_id
    remove_column :pay_subscriptions, :processor

    MODELS.each do |klass|
      remove_column klass.table_name, :processor
      remove_column klass.table_name, :processor_id
      if ActiveRecord::Base.connection.column_exists?(klass.table_name, :pay_data)
        remove_column klass.table_name, :pay_data
      end
      remove_column klass.table_name, :card_type
      remove_column klass.table_name, :card_last4
      remove_column klass.table_name, :card_exp_month
      remove_column klass.table_name, :card_exp_year
      remove_column klass.table_name, :trial_ends_at
    end
  end

  def self.down
    add_column :pay_charges, :owner_type, :string
    add_column :pay_charges, :owner_id, :integer
    add_column :pay_charges, :processor, :string
    add_column :pay_subscriptions, :owner_type, :string
    add_column :pay_subscriptions, :owner_id, :integer
    add_column :pay_subscriptions, :processor, :string

    MODELS.each do |klass|
      add_column klass.table_name, :processor, :string
      add_column klass.table_name, :processor_id, :string
      add_column klass.table_name, :pay_data, Pay::Adapter.json_column_type
      add_column klass.table_name, :card_type, :string
      add_column klass.table_name, :card_last4, :string
      add_column klass.table_name, :card_exp_month, :string
      add_column klass.table_name, :card_exp_year, :string
      add_column klass.table_name, :trial_ends_at, :datetime
    end
  end
end
```

### Pay::Customer

The `Pay::Billable` module has been removed and is replaced with `pay_customer` method on your models.

```ruby
class User
  pay_customer
  pay_merchant
end
```

This adds associations and a couple methods for interacting with Pay.

Instead of adding fields to your models, Pay now manages everything in a `Pay::Customer` model that's associated with your models. You set the default payment processor

```ruby
# Choose a payment provider
user.set_pay_processor :stripe
#=> Creates a Pay::Customer object with associated Stripe::Customer

user.payment_processor
#=> Returns the default Pay::Customer for this user (or nil)

user.pay_customers
#=> Returns all the pay customers associated with this User
```

### Payment Processor

Instead of calling `@user.charge`, Pay 3 moves the `charge`, `subscribe`, and other methods to the `payment_processor` association. This significantly reduces the methods added to the User model.

You can switch between payment processors at anytime and Pay will mark the most recent one as the default. It will also retain the previous Pay::Customers so they can be reused as needed.

```ruby
user.set_pay_processor :stripe

# Charge Stripe::Customer $10
user.payment_processor.charge(10_00)

user.set_payment_processor :braintree
#=> Creates a Pay::Customer object with default: true and associated Braintree::Customer
#=> Updates Pay::Customer for stripe with default: false

user.payment_processor.subscribe(plan: "whatever")
# Subscribes Braintree::Customer to "whatever" plan
# Creates Pay::Subscription record for the subscription
```

### Generic Trials

Generic trials are now done using the fake payment processor

```ruby
user.set_pay_processor :fake, allow_fake: true
user.payment_processor.subscribe(trial_days: 14, ends_at: 14.days.from_now)
user.payment_processor.on_trial? #=> true
```

### Charges & Subscriptions

`Pay::Charge` and `Pay::Subscription` are associated `Pay::Customer` and no longer directly connected to the `owner`

### Payment Methods

Pay 3 now keeps track of multiple payment methods. Each is associated with a Pay::Customer and one is marked as the default. 

We also now support every payment method (previously only Card or PayPal). This means you can store Venmo details, iDeal, FPX, or any other payment method supported by Stripe, Braintree, etc.

To do this, we reformatted the charge and payment method details so they're easier to access:

```ruby
charge.payment_method_type #=> "card"
charge.brand #=> "Visa"
charge.last4 #=> "4242"

charge.payment_method_type #=> "paypal"
charge.email #=> "test@example.org"
```