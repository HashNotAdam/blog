---
title: "Restructuring ViewComponents"
date: 2023-08-23T19:00:00+10:00
slug: "restructuring-viewcomponents"
description: ""
keywords: []
draft: false
tags: [development]
math: false
toc: false
---

I really like working with ViewComponents in Rails. There are so many positives compared to Action View, including:
- a separation of concerns by allowing views to remain presentational, and keeping logic in a view model;
- fast and comprehensive testing;
- a strict interface that requires all parameters to be predefined; and
- 10x better performance than Action View partials.

One thing that has always annoyed me, however, is the file structure. By default, all files are scattered in the same directory, which can be hard to navigate, particularly if you have CSS and JS bundled with your component.

![](/blog/2023/08/restructuring-view-components/default-layout.png)

ViewComponent does offer the ability to sidecar your assets, but you still need to keep the Ruby file outside of the sidecar directory. While this can substantially reduce the clutter, the default ordering in the VS Code Explorer creates a separation between the Ruby file and the sidecar directory.

![](/blog/2023/08/restructuring-view-components/sidecar.png)

My first attempt at making the file structure of ViewComponent feel nicer was to simply change the ordering of the VS Code Explorer, but I soon realised that I consider the default ordering to be far better in most cases; it was only ViewComponent that was causing me problems.

This week I was introduced to the [view_component-contrib gem](https://github.com/palkan/view_component-contrib). One of their features is restructuring ViewComponents so that all related files are in the same directory, and you can drop the term "component" from your components.
Instead of:
```
app/
  components/
    example_component/
      example_component.css
      example_component.html.erb
      example_component.js
    example_component.rb
```
You can have:
```
app/
  components/
    example/
      component.html.erb
      component.rb
      index.css
      index.js
```
While I appreciate the out-of-the-box thinking, I don't like this either. I've encountered issues in the past when lots of files are called the same thing. When searching for a component, even if you include the directory in your search, you'll often need to check the full path to be sure you are about the open the correct file:

![](/blog/2023/08/restructuring-view-components/file-browser.png)

Because Ruby is a dynamically typed language, it is also harder to click a reference to go to the class definition. The modern tooling is getting very good at defaulting the selection to the correct class, but you still need to stop and check the results:

![](/blog/2023/08/restructuring-view-components/show-definition.png)

Instead, I've been experimenting with changing the Rails default, using Components as a namespace, placing all component files in the same directory, but not including that presentational directory in the namespace.

![](/blog/2023/08/restructuring-view-components/restructured.png)

## Zeitwerk

Thankfully, the autoloader used by Rails, Zeitwerk, is quite configurable. To get the outcome I was looking for, I created an initializer, `config/initializers/autoloader.rb`:
```ruby
# frozen_string_literal: true

# The default configuration of View Components can be frustrating. Either you have lots of files
# littered around the app/components directory, or you sidecar your assets but have your Ruby file
# outside of the sidecar.
# In the latter case, the VS Code default is to organise the explorer so that directories are all
# first, followed by files. While this can be changed, the default sort is generally better, but it
# separates the Ruby and asset files.
#
# The following configuration makes it possible for:
# - the components directory to become a namespace;
# - all files related to a component to be grouped in a single directory; and
# - the presentational/grouping directory to be ignored in namespacing.
#
# For example, a component called `ReusableThing` would be defined in:
# `app/components/reusable_thing/reusable_thing.rb`
# resulting in the class:
# `Components::ReusableThing`
#
# Alternatively, you might like to nest/group your components. For example, if you wanted to group
# all components related to forms, you could have:
# `app/components/forms/text_input/text_input.rb`
# resulting in the class:
# `Components::Forms::TextInput`
#
# N.B. this configuration assumes that, if a directory has sub-directories, it should be ignored.
# This means you couldn't do something like:
# - `app/components/forms/text_input/text_input.rb`
# - `app/components/forms/text_input/variation/variation.rb`
# In this case, TextInput would not be collapsed, and so it would be expected that text_input.rb
# defines `Components::Forms::TextInput::TextInput`

# See https://edgeguides.rubyonrails.org/autoloading_and_reloading_constants.html#custom-namespaces
module Components; end

components_dir = Rails.root.join("app/components").to_s.freeze

Rails.autoloaders.main.push_dir(components_dir, namespace: Components)

# Only required for Rails < 7.1
ActiveSupport::Dependencies.autoload_paths.delete(components_dir)
Rails.application.config.watchable_dirs[components_dir] = [:rb]

def collapse_presentational_directories(path)
  children = Dir.children(path)

  if children.any? { File.directory?("#{path}/#{_1}") }
    collapse_children(path, children)
  else
    Rails.autoloaders.main.collapse(path)
  end
end

def collapse_children(path, children)
  children.each do |child|
    child_path = "#{path}/#{child}"
    next unless File.directory?(child_path)

    collapse_presentational_directories(child_path)
  end
end

Dir.children(components_dir).
  find_all { File.directory?("#{components_dir}/#{_1}") }.
  each { collapse_presentational_directories("#{components_dir}/#{_1}") }
```

## Generators

The last piece of the puzzle is to reconfigure the generators to know about the structure, otherwise calling `rails g component ReusableThing` will put the files in the default locations with the default naming structure.

To minimise duplication of effort, I decided to extend upon ViewComponent's AbstractGenerator:
```ruby
# lib/rails/generators/hash_not_adam/abstract_generator.rb
# frozen_string_literal: true

require "rails/generators/abstract_generator"

module HashNotAdam
  module AbstractGenerator
    include ViewComponent::AbstractGenerator

    private

    def destination_directory
      File.join(component_path, class_path, destination_file_name)
    end

    def destination_file_name
      file_name
    end
  end
end
```

Then I created custom generators for the features I use (Ruby/ERB/RSpec).
```ruby
# lib/rails/generators/component/component_generator.rb
# frozen_string_literal: true

require "rails/generators/hash_not_adam/abstract_generator"

module Rails
  module Generators
    class ComponentGenerator < Rails::Generators::NamedBase
      include HashNotAdam::AbstractGenerator

      source_root File.expand_path("templates", __dir__)

      argument :attributes, type: :array, default: [], banner: "attribute"
      check_class_collision

      class_option :inline, type: :boolean, default: false
      class_option :locale, type: :boolean, default: ViewComponent::Base.config.generate.locale
      class_option :parent, type: :string, desc: "The parent class for the generated component"
      class_option :preview, type: :boolean, default: ViewComponent::Base.config.generate.preview
      class_option :stimulus, type: :boolean,
                              default: ViewComponent::Base.config.generate.stimulus_controller

      def create_component_file
        template "component.rb", File.join(destination_directory, "#{file_name}.rb")
      end

      hook_for :test_framework

      hook_for :preview, type: :boolean

      hook_for :stimulus, type: :boolean

      hook_for :locale, type: :boolean

      hook_for :template_engine do |instance, template_engine|
        instance.invoke template_engine, [instance.name]
      end

      private

      def parent_class
        return options[:parent] if options[:parent]

        ViewComponent::Base.config.component_parent_class || default_parent_class
      end

      def initialize_signature
        return if attributes.blank?

        attributes.map { |attr| "#{attr.name}:" }.join(", ")
      end

      def initialize_body
        attributes.map { |attr| "@#{attr.name} = #{attr.name}" }.join("\n    ")
      end

      def initialize_call_method_for_inline?
        options["inline"]
      end

      def default_parent_class
        defined?(ApplicationComponent) ? ApplicationComponent : ViewComponent::Base
      end
    end
  end
end
```

```ruby
# lib/rails/generators/component/templates/component.rb.tt
# frozen_string_literal: true
<% namespaces = class_name.split("::"); last_namespace_index = namespaces.count - 1 %>
module Components
<% namespaces.each_with_index { |namespace, index| %><%=
  " " * ((index + 1) * 2)
%><%=
  if index == last_namespace_index
    "class #{namespace} < #{parent_class}\n"
  else
    "module #{namespace}\n"
  end
%><% } %>
  <%- if initialize_signature -%>
    def initialize(<%= initialize_signature %>)
      <%= initialize_body %>
    end
  <%- end -%>
  <%- if initialize_call_method_for_inline? -%>
    def call
      content_tag :h1, "Hello world!"<%= ", data: { controller: \"#{stimulus_controller}\" }" if options["stimulus"] %>
    end
  <%- end -%>
<% last_namespace_index.downto(0) { |index| %><%=
  " " * ((index + 1) * 2)
%><%=
  "end#{"\n" if index.positive?}"
%><% } %>
end
```

```ruby
# lib/rails/generators/erb/component_generator.rb
# frozen_string_literal: true

require "rails/generators/erb"
require "rails/generators/hash_not_adam/abstract_generator"

module Erb
  module Generators
    class ComponentGenerator < Base
      include HashNotAdam::AbstractGenerator

      source_root File.expand_path("templates", __dir__)
      class_option :inline, type: :boolean, default: false
      class_option :stimulus, type: :boolean, default: false

      def engine_name
        "erb"
      end

      def copy_view_file
        super
      end

      private

      def data_attributes
        if options["stimulus"]
          " data-controller=\"#{stimulus_controller}\""
        end
      end
    end
  end
end
```

```ruby
# lib/rails/generators/erb/templates/component.html.erb.tt
<div<%= data_attributes %>>Add <%= class_name %> template here</div>
```

```ruby
# lib/rails/generators/rspec/component_generator.rb
# frozen_string_literal: true

module Rspec
  module Generators
    class ComponentGenerator < ::Rails::Generators::NamedBase
      source_root File.expand_path("templates", __dir__)

      def create_test_file
        template "component_spec.rb", File.join("spec/components", class_path, "#{file_name}_spec.rb")
      end
    end
  end
end
```

```ruby
# lib/rails/generators/erb/templates/component_spec.rb.tt
# frozen_string_literal: true

require "rails_helper"

RSpec.describe Components::<%= class_name %> do
  pending "add some examples to (or delete) #{__FILE__}"

  # it "renders something useful" do
  #   expect(
  #     render_inline(described_class.new(attr: "value")) { "Hello, components!" }.css("p").to_html
  #   ).to include(
  #     "Hello, components!"
  #   )
  # end
end
```

## Conclusion

This is very much a proof-of-concept that I've only tested at small scale, and I'd love to hear what other people think (especially if you have concerns).

Until I've seen this running for longer, and with many more components, I'm going to be concerned that:
- there could be edge cases I've not considered;
- iterating over the file system to collapse directories might not be efficient at scale; and
- it won't be obvious to anyone new to the project that Zeitwerk has been modified.

I also haven't yet looked at Previews, but I would like to move those into the same directory. This is something that [view_component-contrib](https://github.com/palkan/view_component-contrib) is already doing. While I would want to use a different naming convention, it should be easy enough to reverse-engineer.
