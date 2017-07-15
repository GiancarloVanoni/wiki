## Code Quality Checklist

### Lineamientos generales
* Mantener metodos en 5 lineas o menos
* Mantener clases en 100 lineas o menos 
* Metodos deberian tener 3 variables o menos, preferencia de menos
* Evitar abreviaciones (al nombrar metodos, variables y demas)
* Claridad sobre todo - Facil lectura del codigo - Ser claro y conciso 
* Evitar codigo confuso. KISS - [Keep it simple -S%$%id-](http://principles-wiki.net/principles:keep_it_simple_stupid)
* Buscar metodos publicos que no necesitan serlo y volverlos privados 

* [Manejar exceptions](http://phrogz.net/programmingruby/tut_exceptions.html) de manera agradable/correcta

* Evitar optimizacion prematura. [You aren't gonna need it](https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it)
* No reeinventar la rueda
    - Buscar gemas o librerias que resuelvan lo que queremos hacer
    - Seguir principio [TAM](https://books.google.com/books?id=i6mZ0HBDPzsC&pg=PA214&lpg=PA214&dq=tam+test+tests+activity+maturity&source=bl&ots=pNh9Q-H4SD&sig=fnurOHkQdnx2pj4lHOlIsTGlLBA&hl=en&sa=X&ved=0ahUKEwifmfCYtbnNAhXGSSYKHdd-CjoQ6AEIHDAA#v=onepage&q&f=false) cuando se busca gemas

### Models
* Evitar que los modelos se conviertan en 'fat models' moviendo complejidad a modulos, concerns o classes
* Todos los nombres de modelos y variables deben ser inmediatamente obvios, tan cortos como sea posible y sin usar abreviaciones
* Favorecer composicion sobre herencia
* Usar asociaciones de Active Record [asociaciones](http://guides.rubyonrails.org/association_basics.html) y [finders](http://guides.rubyonrails.org/active_record_querying.html) efectivamente
* Definir [scopes](http://guides.rubyonrails.org/active_record_querying.html#scopes) para queries comunmente usadas
  * Favorecer varios scopes chicos y cortos sobre uno grande y largo 
  * Consolidar cadenas de scopes muy usadas en un mismo scope 
* Usar Active Record [validations](http://guides.rubyonrails.org/active_record_validations.html) para logica de negocio
  * Usar [conditional validations](http://guides.rubyonrails.org/active_record_validations.html#conditional-validation) cuando es apropiado
* Usar Active Record [callbacks](http://guides.rubyonrails.org/active_record_callbacks.html) para logica atada al ciclo de vida de un objeto (e.g. before saving, after saving)
  * Simplificar transacciones largas a traves de callbacks
* Mantener logica de presentacion fuera del modelo


### Controllers
* Manetner controllers thin ('flacos') y lo mas parecido al default scaffold posible
* Usar una o dos variables de instancia por action (cuantas menos mejor)
* Usar [strong parameters](http://edgeguides.rubyonrails.org/action_controller_overview.html#strong-parameters) para seguridad
* Refactorear non-RESTful actions en controller/resource
* Mover logica de negocio a los modelos
* Introducir un[service object](http://railscasts.com/episodes/398-service-objects) cuando el controlador debe coordinar con varios modelos
* No abusar del uso de sesiones. Cuando una sesion es necesaria guardar referencias en vez de instancias

### Views
* No llamar `.find` o `find_by` en las vistas. En cambio para acceder a datos llamar variables de instancia declaradas en el controlador.
* Refactorear en `partials` para hacer las vistas mas reusables
  - pasar variables entre views y partials como locals
* Localizar texto statico y convertirlo. (referencia I18N gem)
* Usar correcta indentacion para html, dos espacios.
* Mover logica simple a helpers. Mover logica compleja a presenters

### Helpers
* No llenar los helpers con metodos innecesarios
* Remover helpers sin metodos
* Agrupar helpers relacionados en presenters
* No tener recursos en helpers. Pasarlos como variables.

### Tests
* Testear unitariamente todo los metodos publicos
* Test de integracion debe fijarse en el comportamiento y no solo en la vista
* Evitar test de vistas y de controladores
* Favorecer [sintaxis de una linea](https://relishapp.com/rspec/rspec-core/docs/subject/one-liner-syntax)
* Evitar uso explicito de `subject`. Usar [subjects con nombre](https://www.relishapp.com/rspec/rspec-core/docs/subject/explicit-subject#use-`subject(:name)`-to-define-a-memoized-helper-method)
* Usar [factories](https://cagit.careerbuilder.com/CorpAppsCB/wiki/wiki/Testing#factory-girl) para preparar data para tests 
* Tests deben correr rapido
* Tener en cuenta que se debe entender un feature solo leyendo sus tests correspondientes
* Usar [shared examples](https://www.relishapp.com/rspec/rspec-core/docs/example-groups/shared-examples) para `DRY` tests
* Usar [predicate matchers](https://www.relishapp.com/rspec/rspec-expectations/v/3-4/docs/built-in-matchers/predicate-matchers) cuando sea posible
* Usar [contexts](http://betterspecs.org/#contexts) para organizar los tests. Empezar su descripcion con `when` or `with`.

### Database
* Usar `seeds.rb` para popular data o crear un [rake task separado](http://rails-4-0.railstutorial.org/book/updating_and_deleting_users#sec-sample_users) para data de ejemplo

### Config
* Asegurarse que Gemfile.lock se commita cuando se añaden o actualizan gemas
* Ser muy cuidadoso con los archivos en 'environments' y sobre todo chequear `production.rb` para asegurar la correcta configuracion

## Code Review Checklist

### Tenemos que ver que de esta parte de CR checklist precisamos y que no

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

## Recursos Adicionales - Rails
* [Rails Antipatterns](http://www.goodreads.com/book/show/9765652-rails-antipatterns)
* [The Rails Way](https://www.amazon.com/Rails-Way-Addison-Wesley-Professional-Ruby/dp/0321944275)
* [Rails Guides](http://guides.rubyonrails.org/)
