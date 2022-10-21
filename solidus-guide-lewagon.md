# Building an E-Commerce Website With Rails

Building a functioning e-commerce website is a gargantuan task. You often need to handle several products, their variants (colours, sizes...), tax categories, shipping methods, payment options, and more. That's why there are several services that offer to do the technological heavy lifting for you, so you can focus on running the business. However, these services can be very restrictive, and make it hard for your site to stand out.

Luckily, there is an alternative. [Solidus](https://github.com/solidusio/solidus) is an e-commerce gem for Ruby on Rails that shares the goal of allowing you to quickly bootstrap your project, while at the same time leave you the flexibility to modify you to your needs as much as you want. In this guide we'll go through how to build an e-commerce site with Solidus and Ruby on Rails.

Due to the level of complexity provided by the Solidus gem, this guide is intended for building a functioning production-level e-commerce site, rather than a (simpler) Demo Day project.

## Versions

Ruby: 3.1.2
Ruby on Rails: 7.0.4
Solidus: 3.2
PostgreSQL: 14.4

## Installing Solidus

Start by creating a new Rails app using the minimal template (don't use the devise template! Solidus comes with its own Devise model):

```
rails new \
  -d postgresql \
  -j webpack \
  -m https://raw.githubusercontent.com/lewagon/rails-templates/master/minimal.rb \
  CHANGE_THIS_TO_YOUR_RAILS_APP_NAME
```

(Solidus doesn't need webpack so you can decide to omit it. At the same time we recommend passing the `-T` flag and using Rspec for tests.)

Next `cd` into the newly created folder, and add and install the gem with the following commands:

```
bundle add solidus
rails g solidus:install --auto-accept=true
```

This will install the gem and run the installer with the default options selected:
- `solidus_auth_devise` for authentication
- `solidus_starter_frontend` for frontend
- an admin user with email `admin@example.com` and password `admin123`

## The Site

You can run the website with `rails s` and access it at `http://localhost:3000/`. You can also access the admin dashboard at `http://localhost:3000/admin` logging in with the admin credentials set during installation.

The initial installation also fills the database with all the necessary entries to do a test run of the checkout process and see some previous orders on the admin panel. You can see a simple demo of the site in the video below:

****video of checkout process****

## Adding a Product

The main modification you'll want to do on the newly created site is adding your own products. But before we get into the HOW to add a product, it's important to understand what a product consists of.

### The Taxonomy of `Spree::Product`

`Spree::Product` is the model name of the product. But when you see a product on the website, a lot of the parts displayed come from other models.

A `Spree::Product` belongs to a `Spree::TaxCategory` (i.e. set of taxes that apply to it) and a `Spree::ShippingCategory` (i.e. shipping options for delivery). It has many `Spree::OptionType`s (i.e. the various selectable options of a product, e.g. size or colour), and each of those has many `Spree::OptionValue`s (e.g. S, M, L for size).

A `Spree::Product` also has many `Spree::Variant`s (i.e. versions of the product), which is comprised of the available combinations of the `Spree::OptionValue`s. And each `Spree::Variant` can have many attached `Spree::Images`.

****screenshot of product with explanation****

So for example you have a Ruby T-Shirt as a `Spree::Product`, with `Spree::Variant`s for S/blue, M/blue, M/red, L/red, etc.

You can find a detailed database diagram of `Spree:Product` and related tables [here](https://dbdiagram.io/d/634fb2c14709410195932aaa).

### Adding a Product Via the Admin Panel

To add a product via the admin panel, simply click on `Products` and then `New Product`. This will lead you to the `/admin/products/new` endpoint. Simply fill out the form and click `Create`.

Once the product has been created, you can click on the `edit` icon to upload images, manage variants, prices, properties, stock, and more.

****screenshot of created item with highlight on edit****

### Adding a Product Via Seeds

Below you can find the snippet of a minimal seed file to add a single product with one image, and a single option type with 3 values.

```ruby
tax_category = Spree::TaxCategory.find_or_create_by!(name: 'Default')
shipping_category = Spree::ShippingCategory.find_or_create_by!(name: 'Default')

puts 'creating products'

chest_armour = Spree::Product.create!(
  name: 'Chest Armour',
  description: 'Lorem Ipsum Description',
  available_on: Time.current,
  tax_category: tax_category,
  shipping_category: shipping_category,
  price: 1599
)

puts 'creating options'

size = Spree::OptionType.create!(name: 'armour-size', presentation: 'Size', position: 1)

small = Spree::OptionValue.create!(name: 'Small', presentation: 'S', position: 1, option_type: size)
medium = Spree::OptionValue.create!(name: 'Medium', presentation: 'M', position: 2, option_type: size)
large = Spree::OptionValue.create!(name: 'Large', presentation: 'L', position: 3, option_type: size)

chest_armour.option_types = [size]
chest_armour.save!

puts 'adding images'

chest_armour.master.images.create!(
  attachment: {
    io: URI.open('https://medievalextreme.com/wp-content/uploads/2020/02/56EEBC5D-7EE1-4114-9FB1-95EF24B4DA85.jpeg'),
    filename: 'brigandine.jpeg'
  }
)

puts 'creating variants'

chest_armour.master.update!(sku: 'CHE-00001')

variants = [
  { product: chest_armour, option_values: [small], sku: 'CHE-00002' },
  { product: chest_armour, option_values: [medium], sku: 'CHE-00003' },
  { product: chest_armour, option_values: [large], sku: 'CHE-00004' }
]

Spree::Variant.create!(variants)
```

## Modifying Views

The default option `solidus_starter_frontend` imports all views into your app folder.

****screenshot of imported views****

To modify views simply change the respective file.

## Modifying Backend Logic

The biggest strength of coding your own e-commerce is in having it work exactly as you'd like. Using a gem to take care of a lot of the complex logic is great, but what is you need to modify some of that logic? You can achieve that through the use of [overrides and Module#prepend](https://guides.solidus.io/customization/customizing-the-core#using-overrides).

(A simple introduction to `Module` is [here](http://ruby-for-beginners.rubymonstas.org/advanced/modules.html). `Module#prepend` is an alternative to `Module#include` which follows the inheritance tree and allows to modify existing code. [Click here](https://medium.com/@leo_hetsch/ruby-modules-include-vs-prepend-vs-extend-f09837a5b073) for a more detailed explanation.)

All the changes will be made in files stored in an `/overrides` folder, so we first need to make sure that Rails loads those files by copy-pasting the following code:

```ruby
# config/application.rb

overrides = "#{Rails.root}/app/overrides"
Rails.autoloaders.main.ignore(overrides)
config.to_prepare do
  Dir.glob("#{overrides}/**/*.rb").each do |override|
    load override
  end
end
```

Then create a file for each logic you want to override. The file will be a set of nested modules, with defined methods (both new and existing ones) at the end of which we'll add `ClassName.prepend(self)`.

For example, normally an order can be canceled if it hasn't been shipped yet (as seen [here](https://github.com/solidusio/solidus/blob/v3.2/core/app/models/spree/order.rb#L274)). But let's say that our business makes bespoke items with a long turnover, and we want to allow a the cancellation of an order only in the first 15 days. We would create an override as follows:

```ruby
# app/overrides/my_store/spree/order/restricted_cancellation.rb

# notice that the path to the file follows the class we are overriding (spree/order)
# and that is also mirrored in the nested modules below

module MyStore
  module Spree
    module Order
      module RestrictedCancellation
        def allow_cancel? # using the same name because we want to override the logic
          created_at > DateTime.current.days_ago(15) && super # check order age AND call default logic for other cases
        end

        ::Spree::Order.prepend(self) # add the custom logic to the existing class
      end
    end
  end
end
```

## Next Steps

This guide serves as entry point for building your own e-commerce, but Solidus offers much more than what was covered here. Be sure to check their [official guides](https://guides.solidus.io/) and if you have any questions check [Stack Overflow](https://stackoverflow.com/questions/tagged/solidus) or drop them a question at their [Slack's #support channel](https://slack.solidus.io/).
