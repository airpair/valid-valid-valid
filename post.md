In this article, you’ll learn some interesting Ruby validation techniques. We’ll drive you trhough input and output validation inside ruby applications, the good and bads and some possible solutions.

You are probably already familiar with the different validation layers present in Rails (or other ruby frameworks). Viewing them from the request point of view we have something like:

![](https://s3.amazonaws.com/wordpress-production/wp-content/uploads/sites/7/2014/11/validation-current-1024x460.png)

Rack validation gives you content type validation and some extra validations like files and body in multipart requests.

Rails versions >4 comes with strong params, which gives you the ability to define which of the input params are required or allowed, and with a not very exhaustive feature you can define the param type choosing between a short list of types.

Model validations in rails come from the `ActiveModel::Validations` module, they are awesome and you can do almost any kind of format, inclusion, uniqueness or custom validation there.

Aside from rails we also have:

* Lotus Validations: http://lucaguidi.com/2014/10/23/introducing-lotus-validations.html

* Sinatra Validations: https://github.com/mattt/sinatra-param

# What is missing

Let’s put it into perspective: if we draw the previous diagram matching sizes with how accurate the validation is in a regular Rails app we will have something like:

![](https://s3.amazonaws.com/wordpress-production/wp-content/uploads/sites/7/2014/11/validation-perspective-1024x371.png)

This makes sense, as you are protecting the most precious attribute of your app, the persisted data. But let’s think about it from the performance and the security point of view: all kind of validation is good for your app health and the sooner you make it the sooner you will be able to stop the request processing, saving machine resources and adding previous layer security in your app.

So, why not move those sizes around and have a bigger param validation layer? why not include format, types, relations and all the power of the `ActiveModel::Validations` in the controller param validator layer?

This idea jumped into our minds during the new API architecture design, and we came up with something called ParamValidators. The idea is dead simple: use plain ruby objects with `ActiveModel::Validations` as service objects to validate and extract params before the controller uses them.

## Param Validators

Let’s start with the base object handling the logic methods to build param validator ruby objects:

https://gist.github.com/andresbravog/7131571668f9aa7192fa

In this basic base object we take care of three things: 

* Collect the defined attributes with `attr_accessor` (allowed ones)

* Initialize them with the given param keys

* Provide a method to return a `HashWithIndifferentAccess` after assignation validation and parsing.

Now we are able to create objects like:

```ruby,linenums=true
# app/validators/param_validator/fancy.rb
 
class ParamValidator::Fancy < ParamValidator::Base
  attr_accessor :user_id, :fancy_name, :fancy_description
  
  validates :user_id, :fancy_name, presence: true
  validates :user_id, integer: true
end
```

Here we are setting some allowed params with `attr_accessor`, some required ones with the presence validator and we could add any other param validator as format, size or type just using the regular `ActiveModel::Validation` methods or any other custom validation build in top of that.

But let’s see how we can use this in the controller. The first thing we built is the ParamsFor module allowing any controller including the module to easily use the param validator objects.

```ruby,linenums=true
# lib/params_for.rb
 
require 'active_support/concern'
 
module ParamsFor
  extend ActiveSupport::Concern
 
  private
  
  # Strong params checker
  #
  # @param name [Symbol] camelcased validator class name
  # @param options [Hash] optional
  # @option options [Boolean] :class class of the validator
  # @return [Hash]
  def params_for(name, options = {})
    instance_var_name = "@#{name.to_s}_params"
    instance_var = instance_variable_get(instance_var_name)
    return instance_var if instance_var.present?
 
    if options[:class]
      validator_klass = options[:class]
    else
      validator_name = "ParamsValidator::#{name.to_s.classify}"
      validator_klass = validator_name.constantize
    end
 
    validator = validator_klass.new(params)
    
    unless validator.valid?
      render status: :bad_request, json: validator.errors.to_json and return 
    end
 
    instance_variable_set(instance_var_name, validator.to_params)
  end
end
```

This is just a meta function allowing you to instantiate a `ParamValidator` object with the current request params, validate it (rendering a :bad_request response with the json errors if is not valid) and memoizing the validated, filtered and parsed Hash. In short, it allows you to easily use `ParamValidator` objects in a controller just like this:

```ruby,linenums=true
# app/controllers/fancy_controller.rb
 
class Fancycontroller < ApplicationController
  include ParamsFor
  
  # Creates a Fancy object by checking and validating params 
  # before that
  #
  def create
    ...
    @fancy = Fancy.new(fancy_params)  
    ...
  end
  
  protected
  
  # Strong params delegated to ParamValidator::Fancy
  # and memoized in @fancy_params var returned by this method
  #
  # @return [HashwithIndifferentAccess]
  def fancy_params
    params_for :fancy  
  end
end
```

## Wrap up

In the last few paragraphs we explored the different validation types and responsibilities in rails applications, understood their use cases and detected the missing parts.

After that we proposed a partial solution to the problem by redifining the architectural way the input validation is built in current rails applications.

## Links

We have an open source project [here](https://github.com/andresbravog/params_for) where you can participate or expose your ideas about the `ParamValidator` thing.

Thanks and happy coding.