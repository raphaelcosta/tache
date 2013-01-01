# Tache
## Just enough Mustache for end users

Tache is a **full Mustache implementation** with the *addition* of "safe" views. Safe views allow Tache templates to be edited by end users without the risk jeopardising your application's security. When using safe views, only explicitly allowed methods are ever invoked and therefore calls to potentially destructive methods such as 'eval' or 'destroy' are ignored.

## Usage

Tache's safe views are an opt-in feature, so Tache will behave just as you'd expect a standard Mustache implementation to behave, unless *you* say otherwise.

### Standard Views

Quick example:

    require 'tache'
    Tache.render('Hello {{planet}}', 'planet' => 'World')
    
...you can guess the result.

Please note that all hash keys must be defined as strings when working with any Tache view. This minimises the risk of memory leaks caused by symbols when used in an end-user environment and is therefore a general requirement of Tache, regardless of whether or not you are using safe views.

Here's another way of doing things (no real departure from standard Mustache):

    class MyView < Tache
      def planet
        'World'
      end
      
      def star
        '*'
      end
      
      def stars
        star * 5
      end
    end
    
    MyView.render("{{planet}} (rating: {{stars}})")
    
Result:

    World (rating: *****)

### Safe Views

In order make use of Tache's safe views, you simply use `Tache::Safe` instead:

    require 'tache/safe'

    Tache::Safe.render('Hello {{planet}}', 'planet' => 'World')
    
The first thing you will notice about safe views (other than them looking a lot like unsafe views), is that only values that have been explicitly exposed via either safe view methods or 'drops' (discussed below) will be invoked. Therefore anything that has not been explicitly exposed will not be invoked or output to the rendered template:

    Tache::Safe.render('Hello {{planet.upcase}}', 'planet' => 'World')
    => "Hello "

Another example, this time subclassing `Tache::Safe`:

    class MySafeView < Tache::Safe
      def thing
        'World'
      end
  
      def present
        'I'm here!'
      end
  
      def bold
        lambda do |text|
          '<b>' + render(text) + '</b>'
        end
      end
    end

Template:

    Hello {{thing}}, here's safe view in action:

    self            -> {{.}}
    inspect         -> {{inspect}}
    to_sym          -> {{to_sym}}
    thing           -> {{thing}}
    present         -> {{present}}
    present.upcase  -> {{present.upcase}}

    Bold: {{#bold}}{{thing}}{{/bold}}

Render:

    MySafeView.render(template)
   
Result:

    Hello World, here's safe view in action:

    self            -> 
    inspect         -> 
    to_sym          -> 
    thing           -> World
    present         -> I'm here!
    present.upcase  -> 

    Bold: <b>World</b>
  
Notice how lambda's still work just as you'd expect but only explicitly exposed values are in the rendered output.

So, what if we want to expose something other than hash values or view object methods to our templates? Well that's a job for Tache Drops.

###Drops

Tache Drops are a concept borrowed from the Liquid template engine. They allow for a way to expose properties of an otherwise unsafe object to a template, safely.

Example:

    class Product
      def title
        'iPhone'
      end
  
      def price
        399
      end
  
      def destroy
        'Deleted product from database!'
      end
  
      # Returns product from database
      def self.first
        Product.new
      end
    end

    class ProductDrop < Tache::Drop
      def initialize(product)
        @product = product
      end
  
      def title
        @product.title
      end
  
      def price
        "$#{@product.price}"
      end
    end

    class CartView < Tache::Safe
      def product
        ProductDrop.new(Product.first)
      end
    end

Template:

    Product price: {{product.price}}
    Trying to destroy: {{product.destroy}}

    {{^destroy}}Bad luck hacker!{{/destroy}}

Render:

    CartView.render(template)

Result:

    Product price: $399
    Trying to destroy: 

    Bad luck hacker!

### Todo

Documentation on compiled templates and partials, coming soon! For now, just see the code (it's pretty straight forward):

    # Compiled
    compiled = MyView.compile('Hello {{thing}}')
    compiled.render
    
    # Partials
    Tache::Safe.render('Hello {{>partial}}', { 'a' => 'b' }, { 'partial' => 'World' })
    
    # Both
    compiled = MyView.compile('Hello {{>partial}}')
    compiled.partials['partial'] = 'World
    compiled.render
    
More on that soon. You can even precompile a punch of partials (they'll get compiled for you automatically anyway but handy to know):

    compiled = MyView.compile('Hello {{>partial1}}, {{>partial2}}')
    compiled.partials = { 
      'partial1' => Tache::Template.compile('{{a.b.c}}'),
      'partial2' => Tache::Template.compile('{{a.b.c.d}}')
    }
    compiled.render

## Installation

Add this line to your application's Gemfile:

    gem 'tache'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install tache

Note: Tache requires at least Ruby 1.9.

## Testing

Tests are written with Test::Unit and can be run with Guard:

    bundle
    bundle exec guard   

## Acknowledgements

Thanks to [Chris Wanstrath](https://github.com/defunkt) for the original Ruby implementation and [Jan Lehnardt](https://github.com/janl) for his JavaScript port (I took a great deal of inspiration, understanding and tests from this version).

Thanks also go out to the guys at [Shopify](https://github.com/Shopify) for the Liquid::Drop inspiration and [Gwendal Roué](https://github.com/groue) for putting up with my brainstorming on GitHub.

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
