## Code Review Checklist
1. CI build passes and is green
  - Ask to make green first
2. No CodeClimate issues
  - Ask to fix first
3. Story exists in VersionOne
  - Review story description, understand scope and review acceptance tests
4. Observe the behavior yourself from end to end
5. Code meets standards of quality based on [Code Quality Checklist](https://cagit.careerbuilder.com/CorpAppsCB/wiki/wiki/Code-Review-Checklist#code-quality-checklist)
6. [GitFlow](https://cagit.careerbuilder.com/CorpAppsCB/authorization_api/wiki/*CA-GitFlow) is followed
  - Branch format
  - Pull request naming
  - Commit message format
  - Commit organization
    - Check need for squashing or reorganization
7. Check if pull request has dependencies with the database or other apis
8. Merge pull request to `develop` branch via GitHub
9. Delete feature branch via GitHub

## Code Quality Checklist

### Lineamientos generales
* Mantener metodos en 5 lineas o menos
* Mantener clases en 100 lineas o menos 
* Metodos deberian tener 3 variables o menos, preferencia de menos
* Evitar abreviaciones (al nombrar metodos, variables y demas)
* Claridad sobre todo - Facil lectura del codigo - Ser claro y conciso 
* Evitar codigo confuso. KISS - [Keep it simple -S%$%id-](http://principles-wiki.net/principles:keep_it_simple_stupid)
* Buscar metodos publicos que no necesitan serlo y volverlos privados 

* Follow [api style guide](https://cagit.careerbuilder.com/CorpAppsCB/api-style-guide) when building APIs

* [Handle exceptions](http://phrogz.net/programmingruby/tut_exceptions.html) gracefully

* Evitar optimizacion prematura. [You aren't gonna need it](https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it)
* No reeinventar la rueda
    - Buscar gemas o librerias que resuelvan lo que queremos hacer
    - Seguir principio [TAM](https://books.google.com/books?id=i6mZ0HBDPzsC&pg=PA214&lpg=PA214&dq=tam+test+tests+activity+maturity&source=bl&ots=pNh9Q-H4SD&sig=fnurOHkQdnx2pj4lHOlIsTGlLBA&hl=en&sa=X&ved=0ahUKEwifmfCYtbnNAhXGSSYKHdd-CjoQ6AEIHDAA#v=onepage&q&f=false) cuando se busca gemas

### Models
* Keep models from becoming fat by moving complexity into modules, concerns and classes
* All model and variable names should be immediately obvious and as short as possible without using abbreviations
* Favor composition over inheritance
* Use Active Record [associations](http://guides.rubyonrails.org/association_basics.html) and [finders](http://guides.rubyonrails.org/active_record_querying.html) effectively
* Define [scopes](http://guides.rubyonrails.org/active_record_querying.html#scopes) for commonly-used queries
  * Favor several small scopes over one large scope 
  * Consolidate common chains of scopes into a single scope 
* Use Active Record [validations](http://guides.rubyonrails.org/active_record_validations.html) for business logic
  * Use [conditional validations](http://guides.rubyonrails.org/active_record_validations.html#conditional-validation) when appropriate
* Use Active Record [callbacks](http://guides.rubyonrails.org/active_record_callbacks.html) for logic tied to an object's lifecycle (e.g. before saving)
  * Simplify large transaction blocks through callbacks
* Keep presentation logic out of the model


### Controllers
* Keep controllers thin and as close to the scaffold as possible
* Use only one or two instance variables per action
* Use [strong parameters](http://edgeguides.rubyonrails.org/action_controller_overview.html#strong-parameters) for security
* Refactor non-RESTful actions into separate controller/resource
* Move business logic to the model
* Introduce a [service object](http://railscasts.com/episodes/398-service-objects) when your controller must coordinate between multiple models
* Don't abuse session. When session is necessary store references instead of instances

### Views
* Avoid calling `.find` or `.find_by` inside views. Rely on instance variables declared in your controller for data.
* Refactor into partials to make views clearer and easier to reuse
  - pass variables in as locals 
* Localize any static text
* Use proper indentation for HTML and indent with 2 spaces
* Move simple view logic into helpers. Move complex view logic into presenters

### Helpers
* Don't pollute helpers with unnecessary methods
* Remove helpers without methods
* Group related helper methods into a presenter
* Don't find resources in helpers. Pass them in as variables.

### Tests
* Unit test all public methods
* Integration tests should focus on behavior and not the view itself
* Every fixed defect should have an associated regression test
* Avoid view and controller tests
* Favor [one-liner syntax](https://relishapp.com/rspec/rspec-core/docs/subject/one-liner-syntax)
* Avoid explicit use of subject. Use a [named subject](https://www.relishapp.com/rspec/rspec-core/docs/subject/explicit-subject#use-`subject(:name)`-to-define-a-memoized-helper-method) instead
* Use [factories](https://cagit.careerbuilder.com/CorpAppsCB/wiki/wiki/Testing#factory-girl) to prepare test data 
* Tests should run fast 
* You should be able to understand a feature by reading its tests 
* Use [shared examples](https://www.relishapp.com/rspec/rspec-core/docs/example-groups/shared-examples) to DRY up tests
* Use [predicate matchers](https://www.relishapp.com/rspec/rspec-expectations/v/3-4/docs/built-in-matchers/predicate-matchers) whenever possible
* Use [contexts](http://betterspecs.org/#contexts) to help organize tests. Start its description with `when` or `with`.

### Database
* Use `seeds.rb` for seed data and create a [separate rake task](http://rails-4-0.railstutorial.org/book/updating_and_deleting_users#sec-sample_users) for sample data
* Use identical casing for SQL Server columns

### Config
* Make sure Gemfile.lock gets committed when adding or updating gems
* Be extra careful with environment files and double-check `production.rb` for the proper configuration

## Additional Resources
* [Rails Antipatterns](http://www.goodreads.com/book/show/9765652-rails-antipatterns)
* [The Rails Way](https://www.amazon.com/Rails-Way-Addison-Wesley-Professional-Ruby/dp/0321944275)
* [Rails Guides](http://guides.rubyonrails.org/)
